+++
title = 'Laravel Unexpected Behavior When Deployed as Subdirectory with NGINX'
date = 2023-10-19T08:50:46+07:00
draft = false
tags = [ "laravel", "nginx" ]
slug = 'laravel-unexpected-behavior-in-nginx'
+++

Years ago I wrote an online course platform using Laravel 6 (now upgraded to 8) for my former employer. Then few days ago, a colleague asking about an error he found when testing that apps. He reached out to me because the bug only happen in the server, but not in his local. Well, that's sound interesting to me.

### The Problem

So, the error goes like this: 
* open a new form page in browser, 
* submit the data, let say POST to `/account-bank`
* when processing, the Laravel app throws an exception 
* caught by my code and passed the message to user:

```
preg_grep(): Unknown identifier 'a'
```

I really didn't expect that error message, because I knew I didn't use `preg_grep()` anywhere. It must be Laravel code. So I inspect it further with `dd()`, here's the code snippet:

```php
public function update(Request $request, BankAccount $accountBank)
    {
        try {
            //...not relevant code omitted
            $data = $request->except('submit');
            $accountBank->fill($data);
            $accountBank->save();
        } catch (\Exception $exception) {
            dd($data, $exception);
            $this->sessionMessage('Account Bank could not be saved. Message: ' . $exception->getMessage(), false);
        }
    }
```

and it led me to this piece of code:

```php
public function isGuarded($key)
    {
        if (empty($this->getGuarded())) {
            return false;
        }

        return $this->getGuarded() == ['*'] ||
               ! empty(preg_grep('/^'.preg_quote($key).'$/i', $this->getGuarded())) ||
               ! $this->isGuardableColumn($key);
    }

//File: vendor/laravel/framework/src/Illuminate/Database/Eloquent/Concerns/GuardsAttributes.php
```

Yep, I was right. The line that called `preg_grep` came from Laravel source. It was checking the key/field name that was about to be filled in when calling `$accountBank->fill($data)`. Then I place `dump($key)` there to know what key caused the error. The result:

{{< figure src="/images/ss-account-bank.png" title="Screenshot of debug output" >}}

Ah, when checking string `/account-bank`, the `/a` prefix was assumed as a modifier/identifier by `preg_grep()`, then throws error because it isn't.

Okay, now we know where the error message come from. But the real question is, why is there `/account-bank` as key with empty value in `$data`? shouldn't it just contain data from request body?

### Finding The Root Cause

#### Context
To understand the cause, I have to give the context on deployment setup. Basically, this apps have 3 main components: 
1. frontend (SPA), 
2. admin web (Laravel), and 
3. backend API (Laravel)

These components deployed using Docker in one server, defined by docker-compose file. Initially, I designed the apps to needs 3 (sub)domains, one for each component. But then someone redo the configuration to only need 1 domain, and made the admin & API as a subdirectory. For example, if we deploy at `e-learning.com`, there'll be `e-learning.com/admin` and `e-learning.com/api` respectively.

As you can guess, we'll need some kind of proxy server in front of the components container. They use Nginx for that purpose. Then if you googling with keyword like `nginx laravel subdirectory`, you may view this [SO post](https://stackoverflow.com/questions/27785372/config-nginx-for-laravel-in-a-subfolder) or this [Github gist](https://gist.github.com/tsolar/8d45ed05bcff8eb75404). The config used was similar to those results:

```nginx
location /admin {
    alias /apps/admin/public;
    try_files $uri $uri/ @admin;
    index index.php;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        fastcgi_pass admin:9000;
        #...other directives
    }
}
location @admin {
    rewrite /admin/(.*)$ /admin/index.php?/$1 last;
}
```
#### Explanation
The culprit is in the last 3 lines, the `rewrite` directive. So, how it affects Laravel behavior is more or less like this:
* to handle request to subdirectory, we define `location` directive
* the `try_files` basically matching the URI request and check if it match a file or directory at root directory, then processing it
* the last value, in this context, was named location called `@admin`. We can say it's a fallback when no matching file found priorly.
* in `@admin`, all request to `/admin/*` was rewritten to `/admin/index.php`, passing the segments after `/admin` as query string
* Bingo! This is the root cause. The URI segments was passed as query string, and Laravel include that as request data with empty value.

#### What really happen
To simulate the behavior, we could trace it like this:
* POST request made to `e-learning.com/admin/account-bank`
* Nginx proxy processing it using `location /admin {...}` config
* The request then passed to `location @admin {...}`
* The request was rewritten as `POST /admin/index.php?/account-bank`
* Laravel process the request, taking `/account-bank` as query string with empty value
* In code, when I use `$request->except(...)`, I got the submitted data from Form, as well as unexpected query string made by Nginx

#### Solution

There are at least 2 possible approach to fix this issue:
1. Update the code
2. Modify Nginx configuration

If we look at code snippet from earlier, I think it's better to refactor the code and implement best practices, e.g. using Form Request and its `validated()` function to fetch the request data. We must assume someone will always try to send weird input, so it's our job to filter those and only fetch what we need.

### Key Takeaways

Some lessons after trial and error trying to make it works as expected:
* always try to use best practices when dealing with input from client side
* debugging in Nginx is not trivial. Although Nginx is popular and well documented, didn't make it obviously easy to understand for every use case.
* be aware of the environment where our code gets deployed. Software developer, especially backend, must have some knowledge about deployment and server. It'll help to debug cases like this.