---
title: Actions 
menuTitle: Actions 
category: API 
position: 9
---

## Motivation

Built in CRUD operations and filtering, Restify allows you to define extra actions for your repositories.

Let's say you have a list of posts and you have to publish them. Usually, for these kind of operations, you have to define a custom route like:

```php
$router->post('posts/publish', PublishPostsController::class);

// PublishPostsController.php

public function __invoke(RestifyRequest $request)
{
  //...
}
```

The `classic` approach is good. However, it has a few limitations. First, you have to manually take care of the `middleware` route, as the testability for these endpoints should be done separately, which might be hard to maintain. Ultimately, the endpoint is disconnected from the repository, which makes it feel out of context so it has a bad readability.

On that wise, code readability, testability and maintainability may become hard.

## Invokable Action Format

The simplest way to define an action is to use the `invokable` class format.

Here's an example:

```php
namespace App\Restify\Actions;

use Illuminate\Http\Request;

class PublishPostAction
{
    public function __invoke(Request $request)
    {
        // $request->input(...)
        
        return response()->json([
            'message' => 'Post published successfully',
        ]);
    }
}
```

Then add the `action` instance to the repository `actions` method:

```php
...
public function actions(RestifyRequest $request): array
{
    return [
        new PublishPostAction,
    ];
}
...
```

Bellow we will see how to define actions in a more advanced way.

## Action definition

The action is nothing more than a class that extends the `Binaryk\LaravelRestify\Actions\Action` abstract class.

It could be generated by using the following command:

```bash
php artisan restify:action PublishPostsAction
```

This will generate the action class:

```php
namespace App\Restify\Actions;

use Binaryk\LaravelRestify\Actions\Action;
use Binaryk\LaravelRestify\Http\Requests\ActionRequest;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Collection;

class PublishPostAction extends Action
{
    public function handle(ActionRequest $request, Collection $models): JsonResponse
    {
        return response()->json();
    }
}
```

The `$models` argument represents a collection of all the models for this query.

### Register action

Then add the action instance to the repository `actions` method:

```php
// PostRepository.php

public function actions(RestifyRequest $request): array
{
    return [
        PublishPostAction::new()
    ];
}
```

### Authorize action

You can authorize certain actions to be active for specific users:

```php
public function actions(RestifyRequest $request): array
{
    return [
        PublishPostAction::new()->canSee(function (Request $request) {
            return $request->user()->can('publishAnyPost', Post::class);
        }),
    ];
}
```

### Call actions

To call an action, you simply access:

```http request
POST: api/restify/posts/actions?action=publish-posts-action
```

The `action` query param value is the `ke-bab` form of the filter class name by default, or a custom `$uriKey` [defined in the action](#custom-uri-key)


The payload could be any type of json data. However, if you're using an [index-action](#index-actions), you are required to pass the `repositories` key, which represents the list of model keys that we apply to this action:

```json
{
  "repositories": [1, 2]
}
```

### Handle action

As soon the action is called, the handled method will be invoked with the `$request` and a list of models matching the keys passed via `repositories`:

```php
public function handle(ActionRequest $request, Collection $models)
{
    $models->each->publish();

    return ok();
}
```

## Action customizations

Actions can be easily customized.

### Action index query

Similarly to repository [index query](/repositories-advanced#index-query), we can do the same by adding the `indexQuery` method on the action:

```php
class PublishPostAction extends Action
{
    public static function indexQuery(RestifyRequest $request, $query)
    {
        $query->whereNotNull('published_at');
    }
    
    ...
}
```

This method will be called right before items are retrieved from the database, so you can filter out or eager load using your custom statements.

### Custom uri key

Since your class names might change along the way, you can define a `$uriKey` property to your actions, so the frontend will always use the same `action` query when applying an action:

```php
class PublishPostAction extends Action
{
    public static $uriKey = 'publish-posts';

    //...

};
```

### Rules

Similarly to [advanced filters rules](/search/advanced-filters#advanced-filter-rules), you could define rules for the action so the payload will get validated before the handle method is fired.

```php
public function rules(): array
{
    return [
        'active' => ['required', 'bool'],
    ];
}
```

<alert type="danger">
Restify doesn't validate the payload automatically as it does for filters, so you're free to validate the payload in the handle method.
</alert>

Always validate the payload as early as possible in the `handle` method:


```php
public function handle(ActionRequest $request, Collection $models)
{
    $request->validate($this->rules());
    
    ...
}
```

## Actions scope

By default, any action could be used on [index](#index-actions) as well as on [show](#show-actions). However, you can choose to instruct your action to be displayed to a specific scope.

## Show actions

Show actions are used when you have to apply them for a single item.

### Show action definition

The show action definition is different, in a way it receives arguments for the `handle` method. 

Restify automatically resolves Eloquent models defined in the route id and passes them to the action's handle method:

```php
// PublishPostAction.php

public function handle(ActionRequest $request, Post $post): JsonResponse
{

}

```

### Show action registration

To register a show action, we have to use the `->onlyOnShow()` accessor:

```php
public function actions(RestifyRequest $request)
{
    return [
        PublishPostAction::new()->onlyOnShow(),
    ];
}
```

### Show action call

The post URL should include the key of the model we want Restify to resolve:

```http request
POST: api/restfiy/posts/1/actions?action=publish-post-action
```

The payload could be empty:

```json
{}
```

### List show actions

To get the list of available actions only for a specific model key:

```http request
GET: api/api/restify/posts/1/actions
```

See [get available actions](#get-available-actions) for more details.

## Index actions

Index actions are used when you have to apply them for a many items.

### Index action definition

The index action definition is different in the way it receives arguments for the `handle` method. 

Restify automatically resolves Eloquent models sent via the `repositories` key into the call payload. Then, it passes it to the action's handle method as a collection of items:

```php
// PublishPostAction.php
use Illuminate\Support\Collection;

public function handle(ActionRequest $request, Collection $posts): JsonResponse
{
    //
}

```

### Index action registration

To register an index action, we have to use the `->onlyOnIndex()` accessor:

```php
// PostRepository.php

public function actions(RestifyRequest $request)
{
    return [
        PublishPostsAction::new()->onlyOnIndex(),
    ];
}
```

### Index action call

The post URL:

```http request
POST: api/restfiy/posts/actions?action=publish-posts-action
```

The payload should always include a key called `repositories`, which is an array of model keys or the `all` keyword if you want to get them all:

```json
{
  "repositories": [1, 2, 3]
}
```

So Restify will resolve posts with ids in the list of `[1, 2, 3]`.

### Apply index action for all

You can apply the index action for all the models from the database if you send the payload: 

```json
{
  "repositories": "all"
}
```

Restify will get chunks of 200 and send them into the `Collection` argument for the `handle` method. 

You can customize the chunk number by customizing the `chunkCount` action property:

```php
// PublishPostAction.php

public static int $chunkCount = 500;
```

### List index actions

To get the list of available actions:

```http request
GET: api/api/restify/posts/actions
```

See [get available actions](#get-available-actions) for more details.

## Standalone actions

Sometimes, you don't need to have an action with models. Let's say for example the authenticated user wants to disable
his/her account. 

### Standalone action definition:

The index action definition is different, in a way it doesn't require the second argument for the `handle`.

```php
// DisableProfileAction.php

public function handle(ActionRequest $request): JsonResponse
{
    //
}

```

### Standalone action registration

There are two ways to register the standalone action:

```php
// UserRepository

public function actions(RestifyRequest $request)
{
    return [
        DisableProfileAction::new()->standalone(),
    ];
}
```

Using the `->standalone()` mutator or by overriding the `$standalone` action property directly into the action:

```php
class DisableProfileAction extends Action
{
    public bool $standalone = true;

    //...
}
```

### Standalone action call

To call a standalone action you're using a similar URL as for the [index action](#index-action-call)

```http request
POST: api/restfiy/users/actions?action=disable-profile-action
```

However, you are not required to pass the `repositories` payload key.

### List standalone actions

Standalone actions will be displayed on both [listing show actions](#list-show-actions) or [listing index actions](#list-index-actions).

## Filters

You can apply any search, match, filter or eager loadings as for a usual request:

```http request
POST: api/api/restify/posts/actions?action=publish-posts-action&id=1&filters=
```

This will apply the match for the `id = 1` and `filter` along with the match for the `repositories` payload you're
sending.

## Action Log

Oftentimes, it is quite useful to view a log of the actions that have been run against a model, or see when the model was
updated, deleted or created (and by whom). 

Thankfully, Restify makes it a breeze to add an action log to a model by attaching the `Binaryk\LaravelRestify\Models\Concerns\HasActionLogs` trait to the repository's corresponding Eloquent model.

### Activate logs

By simply adding the `HasActionLogs` trait to your model, it will log all actions and CRUD operations into the database into the `action_logs` table:

```php
// Post.php

class Post extends Model 
{
    use \Binaryk\LaravelRestify\Models\Concerns\HasActionLogs;
}
```

### Display logs

You can display them by attaching them to the related repository for example:

```php
// PostRepository.php
use Binaryk\LaravelRestify\Fields\MorphToMany;
use Binaryk\LaravelRestify\Repositories\ActionLogRepository;

public static function related(): array
{
    return [
        'logs' => MorphToMany::make('actionLogs', ActionLogRepository::class),
    ];
}
```

Now you can call the posts with logs `api/restify/posts/1?related=logs`, and it will return you the list of actions
performed for posts:

```json
[
  {
    "id": "1",
    "type": "action_logs",
    "attributes": {
      "user_id": "1",
      "name": "Stored",
      "actionable_type": "App\\Models\\Post",
      "actionable_id": "1",
      "status": "finished",
      "original": [],
      "changes": [],
      "exception": ""
    }
  }
]
```

### Custom logs repository

You can definitely use your own `ActionLogRepository`. Just make sure you have it defined into the config: 

```php
// config/restify.php
...
'logs' => [
    'repository' => MyCustomLogsRepository::class,
],
```

## Get available actions

The frontend that consumes your API could check available actions by using this exposed endpoint:

```http request
GET: api/api/restify/posts/actions
```

This will answer with a json like:

```json
{
  "data": {
    "name": "Publish Posts Action",
    "destructive": false,
    "uriKey": "publish-posts-action",
    "payload": []
  }
}
```

`name` - humanized name of the action

`destructive` - you may extend the `Binaryk\LaravelRestify\Actions\DestructiveAction` to indicate to the frontend that
this action is destructive (could be used for deletions)

`uriKey` - is the key of the action and it will be used to perform the action

`payload` - a key / value object indicating required payload defined in the `rules` Action class
