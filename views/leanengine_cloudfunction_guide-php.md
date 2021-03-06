{% extends "./leanengine_cloudfunction_guide.tmpl" %}

{% set platformName = 'PHP' %}
{% set productName = 'LeanEngine' %}
{% set storageName = 'LeanStorage' %}
{% set leanengine_middleware = '[LeanCloud PHP SDK](https://github.com/leancloud/php-sdk)' %}

{% set sdk_guide_link = '[PHP SDK](./leanstorage_guide-php.html)' %}
{% set cloud_func_file = '`$PROJECT_DIR/cloud.php`' %}
{% set runFuncName = '`LeanCloud\Engine\Cloud::run`' %}
{% set defineFuncName = '`LeanCloud\Engine\Cloud::define`' %}
{% set runFuncApiLink = '[LeanCloud\Engine\Cloud::run](/api-docs/php/class-LeanCloud.Engine.Cloud.html#run)' %}

{% set hook_before_save   = "beforeSave" %}
{% set hook_after_save    = "afterSave" %}
{% set hook_before_update = "beforeUpdate" %}
{% set hook_after_update  = "afterUpdate" %}
{% set hook_before_delete = "beforeDelete" %}
{% set hook_after_delete  = "afterDelete" %}
{% set hook_on_verified   = "onVerified" %}
{% set hook_on_login      = "onLogin" %}

{% block cloudFuncExample %}

```php
use \LeanCloud\Engine\Cloud;
use \LeanCloud\Query;
use \LeanCloud\CloudException;

Cloud::define("averageStars", function($params, $user) {
    $query = new Query("Review");
    $query->equalTo("movie", $params["movie"]);
    try {
        $reviews = $query->find();
    } catch (CloudException $ex) {
        // 查询失败, 将错误输出到日志
        error_log($ex->getMessage());
        return 0;
    }
    $sum = 0;
    forEach($reviews as $review) {
        $sum += $review->get("stars");
    }
    if (count($reviews) > 0) {
         return $sum / count($reviews);
    } else {
         return 0;
    }
});
```
{% endblock %}

{% block cloudFuncTimeout %}
### 云函数超时

云引擎超时时间默认为 30 秒，如果超过阈值，进程将被强制 kill：

* 客户端将收到 HTTP status code 为 50X 响应。
* 服务端会出现类似这样的日志：`WARNING: [pool www] child ... exited on signal 9 (SIGKILL) after ... seconds from start`。

{% endblock %}

{% block cloudFuncParams %}
传递给云函数的参数依次为：

* `$params: array`：客户端发送的参数。
* `$user: User`：客户端所关联的用户（根据客户端发送的 `LC-Session` 头）。
* `$meta: array`：有关客户端的更多信息，目前只有一个 `$meta['remoteAddress']` 属性表示客户端的 IP。

{% endblock %}

{% block runFuncExample %}
```php
try {
    $result = Cloud::run("averageStars", array("movie" => "夏洛特烦恼"));
} catch (\Exception $ex) {
    // 云函数错误 
}
```

云引擎中默认会直接进行一次本地的函数调用，而不是像客户端一样发起一个 HTTP 请求。PHP 云引擎暂不支持发起 HTTP 请求来调用云函数。

{% endblock %}

{% block beforeSaveExample %}

```php
Cloud::beforeSave("Review", function($review, $user) {
    $comment = $review->get("comment");
    if ($comment) {
        if (strlen($comment) > 140) {
            // 截断并添加...
            $review->set("comment", substr($comment, 0, 140) . "...");
        }
    } else {
        // 返回错误，并取消数据保存
        throw new FunctionError("No Comment!", 101);
    }
    // 如果正常返回，则数据会保存
});
```
{% endblock %}

{% block afterSaveExample %}

```php
Cloud::afterSave("Comment", function($comment, $user) {
    $query = new Query("Post");
    $post = $query->get($comment->get("post")->getObjectId());
    $post->increment("commentCount");
    try {
        $post->save();
    } catch (CloudException $ex) {
        throw new FunctionError("保存 Post 对象失败: " . $ex->getMessage());
    }
});
```
{% endblock %}

{% block afterSaveExample2 %}

```php
Cloud::afterSave("_User", function($userObj, $currentUser) {
    $userObj->set("from", "LeanCloud");
    try {
        $userObj->save();
    } catch (CloudException $ex) {
        throw new FunctionError("保存 User 对象失败: " . $ex->getMessage());
    }
});
```
{% endblock %}

{% block beforeUpdateExample %}

```php
Cloud::beforeUpdate("Review", function($review, $user) {
    // 对象的 updateKeys 字段记录了本次将要修改的字段名列表，
    // 可用于检测并拒绝对某些字段的修改
    if (in_array("comment", $review->updatedKeys) &&
        strlen($review->get("comment")) > 140) {
        throw new FunctionError("comment 长度不得超过 140 个字符");
    }
});
```

**注意：** 不要修改传入的对象 `$review`，因为对它的改动并不会保存到数据库，但可以抛出异常返回一个错误，拒绝这次修改。
{% endblock %}

{% block afterUpdateExample %}

```php
Cloud::afterUpdate("Article", function($article, $user) {
    // 输出日志到控制台
    error_log("Article {$article->getObjectId()} has been updated.");
});
```
{% endblock %}

{% block beforeDeleteExample %}

```php
Cloud::beforeDelete("Album", function($album, $user) {
    $query = new Query("Photo");
    $query->equalTo("album", $album);
    try {
        $count = $query->count();
    } catch (CloudException $ex) {
        // Delete 操作会被取消
        throw new FunctionError("Error getting photo count: {$ex->getMessage()}");
    }
    if ($count > 0) {
        // 取消 Delete 操作
        throw new FunctionError("Cannot delete album that has photos.");
    }
});
```
{% endblock %}

{% block afterDeleteExample %}

```php
Cloud::afterDelete("Album", function($album, $user) {
    $query = new Query("Photo");
    $query->equalTo("album", $album);
    try {
        // 删除相关的 photos
        $photos = $query->find();
        Object::destroyAll($photos);
    } catch (CloudException $ex) {
        throw new FunctionError("删除关联 photos 失败: {$ex->getMessage()}");
    }
});
```
{% endblock %}

{% block onVerifiedExample %}

```php
Cloud::onVerifed("sms", function($userObj, $meta) {
    error_log("User {$user->getUsername()} verified by SMS");
});
```
{% endblock %}

{% block onLoginExample %}

```php
Cloud::onLogin(function($user) {
    error_log("User {$user->getUsername()} is logging in.");
    if ($user->get("blocked")) {
        // 用户无法登录
        throw new FunctionError("Forbidden");
    }
    // 如果正常执行，则用户将正常登录
});
```
{% endblock %}


{% block hookDeadLoop %}
#### 防止死循环调用

在实际使用中有这样一种场景：在 `Post` 类的 `{{hook_after_update}}` Hook 函数中，对传入的 `Post` 对象做了修改并且保存，而这个保存动作又会再次触发 `{{hook_after_update}}`，由此形成死循环。针对这种情况，我们为所有 Hook 函数传入的 `Object` 对象做了处理，以阻止死循环调用的产生。

不过请注意，以下情况还需要开发者自行处理：

- 对传入的 `Object` 对象进行 `fetch` 操作。
- 重新构造传入的 `Object` 对象，如使用 `Object::create()` 方法。

对于使用上述方式产生的对象，请根据需要自行调用以下 API：

- `Object->disableBeforeHook()` 或
- `Object->disableAfterHook()`

这样，对象的保存或删除动作就不会再次触发相关的 Hook 函数。

```php
Cloud::afterUpdate("Post", function($post, $user) {
    // 直接修改并保存对象不会再次触发 after update hook 函数
    $post->set('foo', 'bar');
    $post->save();

    // 如果有 fetch 操作，则需要在新获得的对象上调用相关的 disable 方法
    // 来确保不会再次触发 Hook 函数
    $post->fetch();
    $post->disableAfterHook();
    $post->set('foo', 'bar');
    $post->save();

    // 如果是其他方式构建对象，则需要在新构建的对象上调用相关的 disable 方法
    // 来确保不会再次触发 Hook 函数
    $post = Object::create("Post", $post->getObjectId());
    $post->disableAfterHook();
    $post->save();
});
```

{% endblock %}
{% block errorCodeExample %}
错误响应码允许自定义。云引擎抛出的 FunctionError（数据存储 API 会抛出此异常）会直接将错误码和原因返回给客户端。若想自定义错误码，可以自行构造 FunctionError，将 code 与 error 传入。否则 code 为 1， message 为错误对象的字符串形式。

```php
Cloud::define("errorCode", function($params, $user) {
    // 尝试登录一个不存在的用户，会返回 211 错误
    User::logIn("not_this_user", "xxxxxx");
});
```
{% endblock %}

{% block errorCodeExample2 %}
```php
Cloud::define("customErrorCode", function($params, $user) {
    // 返回 123 自定义错误信息
    throw new FunctionError("自定义错误信息", 123);
});
```
{% endblock %}

{% block errorCodeExampleForHooks %}
```php
Cloud::beforeSave("Review", function($review, $user) {
   $comment = $review->get("comment");
   if (!$comment) {
       throw new FunctionError(json_encode(array(
           "code" => 123,
           "message" => "自定义错误信息",
       )));
   }
});
```
{% endblock %}

{% block online_editor %}

{% endblock %}

{% block timerExample %}

```php
Cloud::define("logTimer", function($params, $user) {
    error_log("Log in timer");
});
```
{% endblock %}

{% block timerExample2 %}

```php
use \LeanCloud\Push;

Cloud::define("pushTimer", function($params, $user) {
    $push = new Push(array("alert" => "Public message"));
    $push->setChannels(array("Public"));
    $push->send();
});
```
{% endblock %}

{% block masterKeyInit %}
```php
//参数依次为 AppId, AppKey, MasterKey
use \LeanCloud\Client;
Client::initialize($appId, $appKey, $masterKey);
Client::useMasterKey(true);
```
{% endblock %}
