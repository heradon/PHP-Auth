# Auth

Authentication for PHP. Simple, lightweight and secure.

Written once, to be used everywhere.

Completely framework-agnostic and database-agnostic.

## Why do I need this?

 * There are [tons](http://www.troyhunt.com/2011/01/whos-who-of-bad-password-practices.html) [of](http://www.jeremytunnell.com/posts/swab-password-policies-and-two-factor-authentication-a-comedy-of-errors) [websites](http://badpasswordpolicies.tumblr.com/) with weak authentication systems. Don't build such a site.
 * Re-implementing a new authentication system for every PHP project is *not* a good idea.
 * Building your own authentication classes piece by piece, and copying it to every project, is *not* recommended, either.
 * A secure authentication system with an easy-to-use API should be thoroughly designed and planned.
 * Peer-review for your critical infrastructure is *a must*.

## Requirements

 * PHP 5.5.0+
   * PDO
   * OpenSSL

## Installation

 * Set up the PHP library
   * Install via [Composer](https://getcomposer.org/) (recommended)

     `$ composer require delight-im/auth`

     Include the Composer autoloader:

     `require __DIR__.'/vendor/autoload.php';`

   * or
   * Install manually
     * Copy the contents of the [`src`](src) directory to a subfolder of your project
     * Include the files in your code via `require` or `require_once`
 * Set up a database and create the required tables
   * [MySQL](Database/MySQL.sql)

## Usage

### Create a new instance

```php
// $db = new PDO('mysql:dbname=database;host=localhost;charset=utf8', 'username', 'password');
// $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

$auth = new Delight\Auth\Auth($db);
```

If you have an open `PDO` connection already, just re-use it.

If you do enforce HTTPS on your site, pass `true` as the second parameter to the constructor. This is optional and the default is `false`.

Only in the very rare case that you need access to your cookies from JavaScript, pass `true` as the third argument to the constructor. This is optional and the default is `false`. There is almost always a *better* solution than enabling this, however.

If your web server is behind a proxy server and `$_SERVER['REMOTE_ADDR']` only contains the proxy's IP address, you must pass the user's real IP address to the constructor in the fourth argument. The default is `null`.

### Sign up a new user (register)

```php
try {
    $userId = $auth->register($_POST['email'], $_POST['password'], $_POST['username'], function ($selector, $token) {
        // send `$selector` and `$token` to the user (e.g. via email)
    });

    // we have signed up a new user with the ID `$userId`
}
catch (Delight\Auth\InvalidEmailException $e) {
    // invalid email address
}
catch (Delight\Auth\InvalidPasswordException $e) {
    // invalid password
}
catch (Delight\Auth\UserAlreadyExistsException $e) {
    // user already exists
}
catch (Delight\Auth\TooManyRequestsException $e) {
    // too many requests
}
```

The username in the third parameter is optional. You can pass `null` here if you don't want to manage usernames.

For email verification, you should build an URL with the selector and token and send it to the user, e.g.:

```php
$url = 'https://www.example.com/verify_email?selector='.urlencode($selector).'&token='.urlencode($token);
```

If you don't want to perform email verification, just omit the last parameter to `register(...)`. The new user will be active immediately, then.

### Sign in an existing user (login)

```php
try {
    $auth->login($_POST['email'], $_POST['password'], ($_POST['remember'] == 1));

    // user is logged in
}
catch (Delight\Auth\InvalidEmailException $e) {
    // wrong email address
}
catch (Delight\Auth\InvalidPasswordException $e) {
    // wrong password
}
catch (Delight\Auth\EmailNotVerifiedException $e) {
    // email not verified
}
catch (Delight\Auth\TooManyRequestsException $e) {
    // too many requests
}
```

The third parameter controls whether the login is persistent with a long-lived cookie. This is known as the "remember me" feature. Set this to `false` to disable the feature. Otherwise, ask the user if they want to enable "remember me". This is usually done with a checkbox in your user interface. Then use their input to decide between `false` and `true` here. This is optional and the default is `false`.

### Perform email verification

Extract the selector and token from the URL that the user clicked on in the verification email.

```php
try {
    $auth->confirmEmail($_GET['selector'], $_GET['token']);

    // email address has been verified
}
catch (Delight\Auth\InvalidSelectorTokenPairException $e) {
    // invalid token
}
catch (Delight\Auth\TokenExpiredException $e) {
    // token expired
}
catch (Delight\Auth\TooManyRequestsException $e) {
    // too many requests
}
```

### Change the current user's password

If a user is currently logged in, they may change their password.

```php
try {
    $auth->changePassword($_POST['oldPassword'], $_POST['newPassword']);

    // password has been changed
}
catch (Delight\Auth\NotLoggedInException $e) {
    // not logged in
}
catch (Delight\Auth\InvalidPasswordException $e) {
    // invalid password(s)
}
```

### Logout

```php
$auth->logout();

// user has been signed out
```

### Check if the user is signed in

```php
if ($auth->isLoggedIn()) {
    // user is signed in
}
else {
    // user is *not* signed in yet
}
```

A shorthand/alias for this method is `$auth->check()`.

### Get the user's ID

```php
$id = $auth->getUserId();
```

If the user is not currently signed in, this returns `null`.

A shorthand/alias for this method is `$auth->id()`.

### Get the user's email address

```php
$email = $auth->getEmail();
```

If the user is not currently signed in, this returns `null`.

### Get the user's display name

```php
$email = $auth->getUsername();
```

Remember that usernames are optional and there is only a username if you supplied it during registration.

If the user is not currently signed in, this returns `null`.

### Check if the user was "remembered"

```php
if ($auth->isRemembered()) {
    // user did not sign in but was logged in through their long-lived cookie
}
else {
    // user signed in manually
}
```

If the user is not currently signed in, this returns `null`.

### Get the user's IP address

```php
$ip = $auth->getIpAddress();
```

### Utilities

#### Create a random string

```php
$length = 24;
$randomStr = Delight\Auth\Auth::createRandomString($length);
```

#### Create a UUID v4 as per RFC 4122

```php
$uuid = Delight\Auth\Auth::createUuid();
```

## Features

 * registration
   * secure password storage using the bcrypt algorithm
   * email verification through message with confirmation link
   * assurance of unique email addresses
   * customizable password requirements and enforcement
   * optional usernames with customizable restrictions
 * login
   * keeping the user logged in for a long time via secure long-lived token ("remember me")
 * account management
   * change password
   * tracking the time of sign up and last login
   * check if user has been logged in via "remember me" cookie
 * logout
   * full and reliable destruction of session
 * session management
   * protection against session hijacking
   * protection against session fixation attacks
 * throttling
   * per IP address
   * per account
 * enhanced HTTP security
   * prevents clickjacking
   * prevent content sniffing (MIME sniffing)
   * disables caching of potentially sensitive data
 * miscellaneous
   * ready for both IPv4 and IPv6
   * works behind proxy servers as well
   * privacy-friendly (e.g. does *not* save readable IP addresses)

## Exceptions

This library throws two types of exceptions to indicate problems:

   * `AuthException` and its subclasses are thrown whenever a method does not complete successfully. You should *always* catch these exceptions as they carry the normal error responses that you must react to.
   * `AuthError` and its subclasses are thrown whenever there is an internal problem or the library has not been installed correctly. You should *not* catch these exceptions.

## General advice

 * Both serving the authentication pages (e.g. login and registration) and submitting the data entered by the user should only be done over TLS (HTTPS).
 * You should enforce a minimum length for passwords, e.g. 10 characters, but *no* maximum length. Moreover, you should not restrict the set of allowed characters.
 * Whenever a user was remembered ("remember me") and did not log in by entering their password, you should require re-authentication for critical features.
 * Encourage users to use pass*phrases*, i.e. combinations of words or even full sentences, instead of single pass*words*.
 * Do not prevent users' password managers from working correctly. Thus please use the standard form fields only and do not prevent copy and paste.
 * Before executing sensitive account operations (e.g. changing a user's email address, deleting a user's account), you should always require re-authentication, i.e. require the user to sign in once more.
 * You should not offer an online password reset feature ("forgot password") for high-security applications.
 * For high-security applications, you should not use email addresses as identifiers. Instead, choose identifiers that are specific to the application and secret, e.g. an internal customer number.

## Contributing

All contributions are welcome! If you wish to contribute, please create an issue first so that your feature, problem or question can be discussed.

## License

```
Copyright 2015 delight.im <info@delight.im>

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
