# Error handling

A> "Pam, everyone deserves a second second chance"

The C language, which provided the basis for the syntax of many modern languages, has a simple convention for errors. If a function needs to return some data but cannot return due to an error, it returns null. If a function performs some task without returning any result, then in case of success, it returns 0, and in case of error, it returns -1 or some error code. Many PHP developers love this simplicity and use the same principles. The code might look like this:

```php
readonly final class ChangeUserPasswordDto
{
    public function __construct(
        public readonly int $userId, 
        public readonly string $oldPassword, 
        public readonly string $newPassword) 
    {}
}

final class UserService
{
    public function changePassword(
        ChangeUserPasswordDto $dto): bool
    {
        $user = User::find($dto->userId);
        if($user === null) {
            return false; // user not found
        }
        
        if(!password_verify($dto->oldPassword, $user->password)) {
            return false; // old password isn't correct
        }
        
        $user->password = password_hash($dto->newPassword);
        return $user->save();
    }
}

final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        if($service->changePassword($request->getDto())) {
            // return success result
        } else {
            // return error result
        }
    }
}
```
Well, at least it works. But what if the user wants to know the reason for the error? Comments next to `return false` are useless during code execution. We could try error codes, but often, besides the code, additional information is needed for the user. Let's try creating a special class for the function result:

```php
final class FunctionResult
{
    /** @var bool */
    public $success;
    
    /** @var mixed */
    public $returnValue;
    
    /** @var string */
    public $errorMessage;
    
    private function __construct() {}
    
    public static function success(
        $returnValue = null): FunctionResult
    {
        $result = new self();
        $result->success = true;
        $result->returnValue = $returnValue;
        
        return $result;
    }
    
    public static function error(
        string $errorMessage): FunctionResult
    {
        $result = new self();
        $result->success = false;
        $result->errorMessage = $errorMessage;
        
        return $result;
    }
}
```
The constructor of this class is private, so all objects can only be created using the static methods **FunctionResult::success** and **FunctionResult::error**. This simple trick is called "named constructors".

```php
return FunctionResult::error("Something is wrong");
```
Looks much simpler and more informative than

```php
return new FunctionResult(false, null, "Something is wrong");
```

As soon as the constructor of your class becomes so large that calls to new this class look awkward, consider named constructors for it. Our code will look like this:

```php
class UserService
{
    public function changePassword(
        ChangeUserPasswordDto $dto): FunctionResult
    {
        $user = User::find($dto->userId);
        if($user === null) {
            return FunctionResult::error("User was not found");
        }
        
        if(!password_verify($dto->oldPassword, $user->password)) {
            return FunctionResult::error("Old password isn't valid");
        }
        
        $user->password = password_hash($dto->newPassword);
        
        $databaseSaveResult = $user->save();
                
        if(!$databaseSaveResult->success) {
            return FunctionResult::error("Database error");
        }
    
        return FunctionResult::success();
    }
}

final class UserController
{
    public function changePassword(UserService $service, 
                     ChangeUserPasswordRequest $request)
    {
        $result = $service->changePassword($request->getDto());
        
        if($result->success) {
            // return success result
        } else {
            // return error with $result->errorMessage
        }
    }
}
```
Every method (even Eloquent's save() in this imaginary world) returns a **FunctionResult** object with complete information about how the function execution ended. When I showed this example in a workshop, one listener said: "Why do it this way? What about exceptions?" Yes, people invented exceptions, but let me show an example from the Go language:

```go
f, err := os.Open("filename.ext")
if err != nil {
    log.Fatal(err)
}
// Do something with the open *File f
```

Error handling there is implemented similarly, without exceptions. The language's popularity is growing, so it's entirely possible to live without exceptions. However, to continue using the **FunctionResult** class, it will be necessary to implement a function call stack needed for catching errors in the future and correctly logging each error. The entire application will consist of checks **if($result->success)**. It doesn't look like the code of my dreams... I like code that simply describes actions, not checking the error state at every step. Let's try using exceptions.

## Exceptions

When a user asks the application to perform an action (register or cancel an order), the application may or may not be able to do it. In the latter case, there can be many reasons. One of the best illustrations of this is the list of HTTP response codes. There are 2xx and 3xx codes for successful responses, such as 200 Ok or 302 Found. 4xx and 5xx codes are needed for unsuccessful responses, but they differ.

* 4xx for client errors: 400 Bad Request, 401 Unauthorized, 403 Forbidden, etc.
* 5xx for server errors: 500 Internal Server Error, 503 Service Unavailable, etc.

Accordingly, all validation errors, authorization errors, not-found entities, and attempts to change the password with the wrong old password are client errors. Unavailability of an external API, file storage error, or problems connecting to the database are server errors.

There are two opposing schools of error handling:

1. Ascetics of Exception says, "Exceptions are for exceptional cases only." Any exception is considered something quite unusual, able to occur only due to force majeure events (database or file system failure), and almost all exceptions turn into 500 server responses. For situations with incorrectly entered email or wrong password, they use something like a **FunctionResult** object.
2. The adherents of the One True Way school consider any situation that prevents the user's action from being completed an exception.

The code of ascetics looks more logical, but client errors will have to be constantly dragged up, as in the examples above, from functions to those who called them, from the Application Layer to controllers, etc. The code of their opponents has a unified algorithm for dealing with any error (just throw an exception) and cleaner code since there's no need to check the results of methods for errors. There's only one version of the request execution that leads to success:
The application receives valid data.
The user entity was loaded from the database.
The old password matched.
The password is changed to a new one.
Everything is saved to the database.
Any step away from this one true path should cause an exception. The user entered invalid data - exception, this user is not allowed to perform this action - exception, the database server crashed - of course, also an exception. The problem with the One True Way is that exceptions should be separated into client and server errors somewhere since we need to generate different responses (remember about 400 and 500 response codes?), and such errors should be logged differently.

It's hard to say which path is preferable. The second path seems more appealing when an application has just acquired a separate Application Layer. The code is cleaner. An exception can be thrown anywhere, in any private service class method, if something is wrong, and it will immediately reach its destination. However, if the application continues to grow, for example, a Domain Layer is also created, then this fascination with exceptions may prove harmful. Some of them, being thrown but not caught at the right level, can be misinterpreted at a higher level. The number of try-catch blocks will increase, and the code will no longer be so clean.

Laravel throws exceptions for a 404 error, for access error (code 403), and in general, has a **HttpException** class where you can specify the HTTP error code. Therefore, I will also choose the second option in this book and generate exceptions for any problems.

Writing code with exceptions:

```php
class UserService
{
    public function changePassword(
        ChangeUserPasswordDto $dto): void
    {
        $user = User::findOrFail($dto->userId);
        
        if(!password_verify($dto->oldPassword, $user->password)) {
            throw new \Exception("Old password is not valid");
        }
        
        $user->password = password_hash($dto->newPassword);
        
        $user->saveOrFail();
    }
}

final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        try {
            $service->changePassword($request->getDto());
        } catch(\Throwable $e) {
            // log error
            // return failure web response with $e->getMessage();
        }
        
        // return success web response
    }
}
```
Even with such a simple example, it's evident that the code of the method **UserService::changePassword** has become much cleaner. Any deviation from the main execution path triggers an exception caught in the controller. Eloquent also has methods for working in this style: **findOrFail()**, **firstOrFail()**, and some other ***OrFail()** methods. However, this code still has problems:

1. **Exception::getMessage()** is not the best message to show for a user. The message "Old password is not valid" is okay, but, for example, "Server Has Gone Away (error 2006)" definitely is not.
2. Any server errors should be logged. Small applications use log files. When the application becomes popular, exceptions can occur every second. Some exceptions signal problems in the code and should be fixed immediately. Some exceptions are expected: the internet is imperfect, and requests to even the most stable APIs can fail once a million times. However, developers should also react if the frequency of such errors suddenly increases (some external API stopped working). In such cases, when error control requires much attention, it's better to use specialized services that allow grouping exceptions and working with them much more conveniently. If interested, you can google "error monitoring services" and find several such services. Large companies build specialized solutions for recording and analyzing logs from all their servers. Some companies do not log client errors. Some log them but in separate storages. In any case, it's crucial to separate server and client errors for any application.

## Base exception class

The first step is to create a base class for all business logic exceptions, such as "Old password is not valid". There is a **\DomainException** class in PHP that could be used for this purpose, but it is already used in other places, for example, in third-party libraries, which can lead to confusion. It's easier to create a new class, say, **BusinessException**.

```php
class BusinessException extends \Exception
{
    /**
    * @var string 
    */
    private $userMessage;
    
    public function __construct(string $userMessage)
    {
        $this->userMessage = $userMessage;
        parent::__construct("Business exception");
    }
    
    public function getUserMessage(): string
    {
        return $this->userMessage; 
    }
}

// Password verification error now will throw an exception

if(!password_verify($command->getOldPassword(), $user->password)) {
    throw new BusinessException("Old password is not valid");
}

final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        try {
            $service->changePassword($request->getDto());
        } catch(BusinessException $e) {
            // Return error response
            // with one of 400x HTTP code
            // and $e->getUserMessage();
        } catch(\Throwable $e) {
            // log exception
            
            // return error response
            // With 500 HTTP code
            // and something like "Houston, we have a problem"
        }
        
        // return success response
    }
}
```
This code catches **BusinessException** and displays its message to the user. Other exceptions will show some "Internal error, we are working on it" message and the exception will be sent to the log. The code works correctly, but the **catch** section will be identically repeated in every method of every controller. It makes sense to move the exception-handling logic to a higher level.

## Global handler

In Laravel (as in almost all frameworks), there is a global exception handler, and surprisingly, it's very convenient to handle almost all exceptions of our application here. In newer versions of Laravel, it is structured differently. I will look at the **app/Exceptions/Handler.php** class from older versions. The **Handler** class implements two closely related responsibilities: logging exceptions and informing users about them.

```php

namespace App\Exceptions;

class Handler extends ExceptionHandler
{
    protected $dontReport = [
        // It means BusinessException
        // won't be logged
        // but will be shown to users
        BusinessException::class,
    ];

    public function report(Exception $e)
    {
        if ($this->shouldReport($e))
        {
            // This is a good place for
            // 3rd party error monitoring
            // services integration
        }

        // This will log exception
        // to storage/logs/laravel.log
        parent::report($e);
    }

    public function render($request, Exception $e)
    {
        if ($e instanceof BusinessException)
        {
            if($request->ajax())
            {
                $json = [
                    'success' => false,
                    'error' => $e->getUserMessage(),
                ];

                return response()->json($json, 400);
            }
            else
            {
                return redirect()->back()
                       ->withInput()
                       ->withErrors([
                           'error' => trans($e->getUserMessage())]);
            }
        }

        // Standard error page
        return parent::render($request, $e);
    }
}
```
A simple example of a global handler. The **report** method can be used for additional logging. The entire **catch** section from the controller has moved to the **render** method. Here, all logic errors will be caught, and the correct messages for the user will be generated. Look at the controller:

```php
final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        $service->changePassword($request->getDto());
                
        // return success result
    }
}
```

Excellent! The business logic has moved from the controller to the service class. Validation is in the Request object. Exception handling is in the global handler. The controller is left to control the process at the highest level. Finally, its work matches its name!

## Checked and unchecked exceptions

Close your eyes. I'm now going to talk about lofty matters that will ultimately prove to be useless. Imagine the seashore and the **UserService::changePassword** method.
What errors might occur there?

* **Illuminate\Database\Eloquent\ModelNotFoundException** if a user with such **id** does not exist
* **Illuminate\Database\QueryException** if the database query cannot be executed
* **App\Exceptions\BusinessException** if the old password is incorrect
* **TypeError** if somewhere deep inside the code the function **foo**(**SomeClass** **$x**) receives a parameter **$x** of a different type
* **Error** if **$var->method()** is called when variable **$var** == **null**
* and many other exceptions

From the caller's point of view, some errors, like **Error**, **TypeError**, **QueryException**, are absolutely out of context. Some HTTP controller doesn't even know what to do with these errors. It can only show the user a message "Something bad happened, and I don't know what to do with it". But some of them make sense to it. **BusinessException** indicates something is wrong with the logic, and there are messages directly for the user, and the controller definitely knows what to do with this exception. The same can be said about **ModelNotFoundException**. The controller can show a 404 error for it. So, two types of errors:

1. Errors that are understandable to the calling code and can be effectively handled there.
2. Other errors.

It would be good to handle the first errors where this method is called, while the second type can be thrown up further. Remember this, and let's look at the Java language.

```java
public class Foo
{
    public void bar()
    {
        throw new Exception("test");
    }
}
```
This code won't even be compiled.
Compiler will say: "Error:(5, 9) java: unreported exception java.lang.Exception; must be caught or declared to be thrown"
There are two ways to fix that. Catch it:

```java
public class Foo
{
    public void bar()
    {
        try {
            throw new Exception("test");
        } catch(Exception e) {
            // do something
        }
    }
}
```
Or describe it in the method signature:

```java
public class Foo
{
    public void bar() throws Exception
    {
        throw new Exception("test");
    }
}
```
In that case, every **Foo::bar** method caller should do the same - catch it or add it to the **throws**:

```java
public class FooCaller
{
    public void caller() throws Exception
    {
        (new Foo)->bar();
    }
    
    public void caller2()
    {
        try {
            (new Foo)->bar();
        } catch(Exception e) {
            // do something
        }
    }
}
```
Of course, handling **every** exception in such a detailed manner would be quite a torment. In Java, exceptions are divided into two types:
1. **Checked** exceptions must be caught or declared in the method's signature.
2. **Unchecked** exceptions can be thrown without additional declarations.

Let's look at the root of the exception class hierarchy in Java (PHP, starting from version 7, has the same one):

```
          Throwable(checked)
         /         \
Error(unchecked)  Exception(checked)
                        \
                      RuntimeException(unchecked)
```
**Throwable**, **Exception** and their descendants - checked exceptions.
Except **Error**, **RuntimeException** and their descendants. They can be thrown anywhere without any conditions.

Returning to the two types of errors:

1. Errors that are understandable to the calling code and can be effectively handled there.
2. Other errors.

Checked exceptions are designed for the first type of error, while unchecked exceptions are for the second type. The calling code can effectively handle a checked exception, and this strictness obligates it to do so. This leads to more correct error handling.

Alright, Java has this, PHP doesn't. But the IDE I use, PhpStorm, simulates the behavior of Java.
```php
class Foo
{
    public function bar()
    {
        throw new Exception();
    }
}
```
PhpStorm highlights 'throw new Exception();' with warning: 'Unhandled Exception'.
There are two ways to handle this:

1. Catch the exception
2. Describe it in the @throws phpDoc-comment: 

```php
class Foo
{
    /**
     * @throws Exception
     */
    public function bar()
    {
        throw new Exception();
    }
}
```

The list of unchecked exceptions is configurable. By default, it is: **\Error**, **\RuntimeException**, and **\LogicException**. They can be thrown without warnings.

With all this information, a developer can design an exception class structure for an application. I want to inform the code calling **UserService::changePassword** about the following errors:

1. **ModelNotFoundException**, when a user with such **id** is not found
2. **BusinessException**: this error contains a message intended for the user and can be handled immediately.
   
All other errors can be handled later.
So, in an ideal world:

```php
class ModelNotFoundException extends \Exception
{...}

class BusinessException extends \Exception
{...}

final class UserService
{
    /**
     * @param ChangeUserPasswordDto $command
     * @throws ModelNotFoundException
     * @throws BusinessException
     */
    public function changePassword(
        ChangeUserPasswordDto $command): void
    {...}
}
```
But since we have already moved all the error-handling logic to a global handler, we would need to replicate all these **@throws** tags in the controller method:

```php
final class UserController
{
    /**
     * @param UserService $service
     * @param Request $request
     * @throws ModelNotFoundException
     * @throws BusinessException
     */
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        $service->changePassword($request->getDto());
                
        // return success response
    }
}
```
Even though PhpStorm can automatically generate all these tags, it's not convenient. Returning to our imperfect world: The **ModelNotFoundException** class in Laravel is already inherited from **\RuntimeException**. Accordingly, it is unchecked by default. This makes sense because deep within its error handling, Laravel processes these exceptions itself. Therefore, in our current situation, it would also be wise to make such a concession with conscience:

```php
class BusinessException extends \RuntimeException
{...}
```

And forget about the **@throws** tags, keeping in mind that all **BusinessException** exceptions will be handled in the global handler.

This is one of the main reasons why new languages do not have the feature of checked exceptions, and many Java developers hate them. Another reason is that some libraries simply write "throws Exception" in their methods. "throws Exception" does not provide any helpful information. It just forces the client code to repeat this useless "throws Exception" in its signature.

I will return to exceptions in the Domain Layer chapter when this approach with unchecked exceptions becomes inconvenient.

## A few words at the end of the chapter

A function or method that returns more than one type, can return **null**, or returns a boolean value (whether everything went well or not), makes the calling code less clean. The return value will need to be checked immediately after calling. Code with exceptions looks much cleaner:

```php
// Without exceptions
$user = User::find($command->getUserId());
if($user === null) {
    // handle the error
}

$user->doSomething();


// With exceptions
$user = User::findOrFail($command->getUserId());
$user->doSomething();
```

On the other hand, using objects like **FunctionResult** gives developers better control over execution. For example, **findOrFail** called at the wrong place at the wrong time will force the application to show the user a 404 error instead of a correct error message. So, it is better always to be cautious with exceptions.
