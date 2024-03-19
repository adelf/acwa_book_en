# Events

The action of the Application Layer always contains a main part where the core logic of the action is performed, as well as some additional actions. User registration consists of the actual creation of a user entity and the sending of a corresponding letter. Updating the text of an article involves updating the **$post->text** value with the entity's persisting to the database, as well as, for example, a **Cache::forget** call to invalidate the cache. Here is a pseudo-real example: a survey website. Creating a survey entity:

```php
final class SurveyService
{
    public function __construct(
        private ProfanityFilter $profanityFilter
        /*, ... other dependencies */
    ) {}
    
    public function create(SurveyCreateDto $request)
    {
        $survey = new Survey();
        $survey->question = $this->profanityFilter->filter(
            $request->getQuestion());
        //...
        $survey->save();
        
        foreach($request->getAnswers() as $answerText)
        {
            $answer = new SurveyAnswer();
            $answer->survey_id = $survey->id;
            $answer->text = 
                    $this->profanityFilter->filter($answerText);
            $answer->save();
        }
        
        // sitemap generation
        
        // make an API-call (like webhook)
    }
}
```

This is a typical survey creation with answer options, filtering profanity in all texts, and some post-actions. The survey object is complex: it's useless without answer options. We must ensure its consistency. That brief moment when we've created the survey object and added it to the database but haven't created its answer options yet is very dangerous!

## Database transactions

The first problem is the database. Any exception between objects persisting and the survey object will be in the database without any answer options. All database engines designed for storing critical data have a transaction mechanism. They guarantee consistency within a transaction. All queries wrapped in a transaction will be fully completed, or if an error occurs during execution, none will. This looks like a solution:

```php
final class SurveyService
{
    public function __construct(
        ..., private DatabaseConnection $connection
    ) {}
 
    public function create(SurveyCreateDto $request)
    {
        $this->connection->transaction(function() use ($request) {
            $survey = new Survey();
            $survey->question = $this->profanityFilter->filter(
                    $request->getQuestion());
            //...
            $survey->save();
            
            foreach($request->getOptions() as $optionText) {
                $option = new SurveyOption();
                $option->survey_id = $survey->id;
                $option->text = 
                    $this->profanityFilter->filter($optionText);
                $option->save();
            }
            
            // sitemap generation
        
            // make an API-call (like webhook)    
        });
    }
}
```

Our data are consistent now, but the magic of transactions comes at a cost to the database. When executing queries within a transaction, the database needs to maintain two sets of data: for a successful outcome and for an unsuccessful one. This can significantly impact performance for projects that might perform hundreds of simultaneous transactions. For projects with a lower load, it's not as critical, but it's still a good habit to complete transactions as quickly as possible. Profanity checking might require a call to a special API, which can take considerable time. Let's try to move everything possible out of the transaction:

```php
final class SurveyService
{
public function create(SurveyCreateDto $request)
{
    $filteredRequest = $this->filterRequest($request);
    
    $this->connection->transaction(
        function() use ($filteredRequest) {
            $survey = new Survey();
            $survey->question = $filteredRequest->getQuestion();
            //...
            $survey->save();
            
            foreach($filteredRequest->getOptions() 
                                as $optionText) {
                $option = new SurveyOption();
                $option->survey_id = $survey->id;
                $option->text = $optionText;
                $option->save();
            }
        });
    
    // sitemap generation
        
    // make an API-call (like webhook)    
}

private function filterRequest(
    SurveyCreateDto $request): SurveyCreateDto
{
    // filters the texts in request
    // and returns the same DTO, but with cleared data
}
}
```

Now, only light actions are inside the transaction, and it is executed instantly. This is how it should be.

## Queues

The second problem is the query execution time. The application should respond as quickly as possible. Creating a survey involves heavy actions, such as generating a sitemap or calling external APIs. The usual solution is to defer these actions using a queuing mechanism. Instead of performing a heavy action in the web request handler, the application can create a task for performing this action and put it in a queue. Then, a service on the same server will pick it up from the queue. Or it will be on other servers, specifically created to process tasks from queues, known as worker servers. The queue can be a table in the database, a list in Redis, or special queue software like Kafka or RabbitMQ.

Laravel provides several ways to work with queues, one of which is jobs. As I mentioned, an action consists of a main part and several secondary actions. The main action in creating a survey entity cannot be done without profanity filtering, but all post-actions can be deferred. In fact, in some situations where an action takes too long, you can completely defer it, telling the user, "We will complete it soon," but that's not our case yet.

```php
final class SitemapGenerationJob implements ShouldQueue
{
    public function handle()
    {
        // sitemap generation
    }
}

final class NotifyExternalApiJob implements ShouldQueue {}

use Illuminate\Contracts\Bus\Dispatcher;

final class SurveyService
{
    public function __construct(..., 
        private Dispatcher $dispatcher
    ) {}
        
    public function create(SurveyCreateDto $request)
    {
        $filteredRequest = $this->filterRequest($request);
        
        $survey = new Survey();
        $this->connection->transaction(...);
        
        $this->dispatcher->dispatch(
            new SitemapGenerationJob());
        $this->dispatcher->dispatch(
            new NotifyExternalApiJob($survey->id));
    }
}
```

If a class implements the **ShouldQueue** interface, the execution of that task will be deferred to a queue. This code executes quickly enough, but I still don't like it. There can be a multitude of post-actions, and the service class begins to know too much. It executes each post-action, but from a high-level perspective, creating a survey should not be about generating a sitemap or interacting with external APIs. Controlling projects with a vast number of post-actions becomes very challenging.

## Events

Instead of directly calling each post-action, the service class can simply inform the application that something has happened. The application can respond to these events by performing the necessary post-actions. Laravel supports an event mechanism:

```php
final class SurveyCreated
{
    public function __construct(
        public readonly int $surveyId
    ) {}
}

use Illuminate\Contracts\Events\Dispatcher;

final class SurveyService
{
    public function __construct(..., 
        private Dispatcher $dispatcher
    ) {}

    public function create(SurveyCreateDto $request)
    {
        // ...
        
        $survey = new Survey();
        $this->connection->transaction(
            function() use ($filteredRequest, $survey) {
                // ...
            });
        
        $this->dispatcher->dispatch(new SurveyCreated($survey->id));
    }
}

final class SitemapGenerationListener implements ShouldQueue
{
    public function handle($event)
    {
        // sitemap generation
    }
}

final class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        SurveyCreated::class => [
            SitemapGenerationListener::class,
            NotifyExternalApiListener::class,
        ],
    ];
}
```

Now, the Application Layer simply announces that a new survey has been created (the **SurveyCreated** event). The application is configured to respond to events. Event listener classes (**Listeners**) contain reactions to events. The **ShouldQueue** interface works just the same, indicating when the execution of this listener should start: immediately or deferred. Events are a very powerful concept, but there are also pitfalls here.

## Using Eloquent events

Laravel generates tons of events. Cache system events: **CacheHit**, **CacheMissed**, etc. Notification events: **NotificationSent**, **NotificationFailed**, etc. Eloquent also generates its events. Here's an example from the documentation:

```php

class User extends Authenticatable
{
    /**
     * The event map for the model.
     *
     * @var array
     */
    protected $dispatchesEvents = [
        'saved' => UserSaved::class,
        'deleted' => UserDeleted::class,
    ];
}
```

The **UserSaved** event will be generated whenever a user entity is saved to the database. Being saved means any update or insert query. Using these events comes with several disadvantages.

**UserSaved** is not the correct name for this event. **UsersTableRowInsertedOrUpdated** would be more appropriate. But even that isn't always correct. This event won't be generated during bulk operations with database rows. The "deleted" event won't be triggered if a row in the database is removed using the database's cascade delete mechanism. The main issue, however, is that these are infrastructure-level events, database row events, but they are used as business events or domain events. The difference is easy to grasp with the example of creating a survey entity:

```php
final class SurveyService
{
public function create(SurveyCreateDto $request)
{
    //...
    $this->connection->transaction(function() use (...) {
        $survey = new Survey();
        $survey->question = $filteredRequest->getQuestion();
        //...
        $survey->save();
        
        foreach($filteredRequest->getOptions() as $optionText){
            $option = new SurveyOption();
            $option->survey_id = $survey->id;
            $option->text = $optionText;
            $option->save();
        }
    });
    //...
}
}
```

Calling **$survey->save();** generates a 'saved' event for that entity. The first problem is that the survey object is not yet ready and is inconsistent. It still lacks answer options. A listener wanting to send an email with this survey would naturally want the object to be complete, with all its answer options. This isn't a problem for deferred listeners, but we can't be sure about our listeners in the Application layer code. I strongly recommend avoiding code that works correctly "sometimes, under certain conditions" but may occasionally deliver an unpleasant surprise.

The second problem is that these events are triggered directly within the transaction. Executing listeners immediately or even queuing them makes transactions longer and more fragile. The worst part is that an event like **SurveyCreated** could be triggered, but an error may occur later in the transaction, causing it to be rolled back. Nonetheless, the email to the user about a survey that wasn't even created will still be sent. I found a couple of Laravel packages that catch all these events, temporarily store them, and only execute them after the transaction has been successfully completed (search for "Laravel transactional events"). Yes, they solve many of these problems, but it all seems so unnatural! The simple idea of generating a standard business event **SurveyCreated** after a successful transaction is much better.

## Entities as event class fields
I often see Eloquent entities used directly in event fields:

```php
final class SurveyCreated
{
    public function __construct(
        public readonly Survey $survey
    ) {}
}

final class SurveyService
{
    public function create(SurveyCreateDto $request)
    {
        // ...
        $survey = new Survey();
        // ...
        $this->dispatcher->dispatch(new SurveyCreated($survey));
    }
}

final class SendSurveyCreatedEmailListener implements ShouldQueue
{
    public function handle(SurveyCreated $event)
    {
        // ...
        foreach($event->survey->options as $option)
        {...}
    }
}
```
This is a straightforward example of a listener that utilizes the values from a **HasMany** relationship. This code works. When the code **$event->survey->options** is executed, Eloquent makes a database query and fetches all the answer options. Here's another example:

```php
final class SurveyOptionAdded
{
    public function __construct(
        public readonly Survey $survey
    ) {}
}

final class SurveyService
{
public function addOption(SurveyAddOptionDto $request)
{
    $survey = Survey::findOrFail($request->getSurveyId());
    
    if($survey->options->count() >= Survey::MAX_POSSIBLE_OPTIONS) {
        throw new BusinessException('Max options amount exceeded');
    }

    $survey->options()->create(...);
    
    $this->dispatcher->dispatch(new SurveyOptionAdded($survey));
}
}

final class SomeListener implements ShouldQueue
{
    public function handle(SurveyOptionAdded $event)
    {
        // ...
        foreach($event->survey->options as $option)
        {...}
    }
}
```

In this case, things aren't as smooth. When the service class checks the number of answer options, it retrieves a fresh collection of the answer options for the survey. Then, it adds a new answer option by calling **$survey->options()->create(...);**. Afterward, a listener executing **$event->survey->options** receives an outdated version of the answer options, excluding the newly created one. This happens due to Eloquent's two mechanisms for handling relationships: the **options()** method and the **options** pseudo-field, which corresponds to this method but maintains its own version of the data. Therefore, when passing an entity to events, developers should ensure the consistency of relationship values, for example, by calling:

```php
$survey->load('options');
```

before passing the object into the event. This requirement makes the application code quite fragile. It's easily broken by carelessly passing the object into an event. It's much simpler to just pass the entity's **id**. Each listener can always fetch the latest version of the entity by querying the database with this **id**.

## A few words at the end of the chapter

As the application develops, especially with increasing server load, there's a natural desire to reduce server response and database transaction execution time.

All heavy actions are better performed in another thread, typically using queues. This is not inherently complex; it's entirely feasible with organized and meticulous planning. In complex cases, when the number of actions and post-actions becomes large, events and setting up their listeners help organize everything without turning the application code into chaos.

There's also no inherent complexity with transactions, but implicit and hidden logic, such as Eloquent model events, can introduce confusion into the process. Explicit events, triggered in controlled places, are a much more stable and manageable strategy. An event like **UserBanned** clearly expresses what happened, unlike the much more generic **UserSaved**.