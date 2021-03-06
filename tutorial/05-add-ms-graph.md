<!-- markdownlint-disable MD002 MD041 -->

在本练习中，将把 Microsoft Graph 合并到应用程序中。 对于此应用程序，您将使用 [microsoft graph](https://github.com/microsoftgraph/msgraph-sdk-php) 库对 microsoft graph 进行调用。

## <a name="get-calendar-events-from-outlook"></a>从 Outlook 获取日历事件

1. 在 **/app** 目录中创建一个名为的新目录 `TimeZones` ，然后在名为的该目录中创建一个新文件， `TimeZones.php` 并添加以下代码。

    :::code language="php" source="../demo/graph-tutorial/app/TimeZones/TimeZones.php":::

    此类实现 Windows 时区名称到 IANA 时区标识符的简化映射。

1. 在 " **/app/Http/Controllers** " 目录中创建一个名为的新文件 `CalendarController.php` ，并添加以下代码。

    ```php
    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;
    use Microsoft\Graph\Graph;
    use Microsoft\Graph\Model;
    use App\TokenStore\TokenCache;
    use App\TimeZones\TimeZones;

    class CalendarController extends Controller
    {
      public function calendar()
      {
        $viewData = $this->loadViewData();

        $graph = $this->getGraph();

        // Get user's timezone
        $timezone = TimeZones::getTzFromWindows($viewData['userTimeZone']);

        // Get start and end of week
        $startOfWeek = new \DateTimeImmutable('sunday -1 week', $timezone);
        $endOfWeek = new \DateTimeImmutable('sunday', $timezone);

        $queryParams = array(
          'startDateTime' => $startOfWeek->format(\DateTimeInterface::ISO8601),
          'endDateTime' => $endOfWeek->format(\DateTimeInterface::ISO8601),
          // Only request the properties used by the app
          '$select' => 'subject,organizer,start,end',
          // Sort them by start time
          '$orderby' => 'start/dateTime',
          // Limit results to 25
          '$top' => 25
        );

        // Append query parameters to the '/me/calendarView' url
        $getEventsUrl = '/me/calendarView?'.http_build_query($queryParams);

        $events = $graph->createRequest('GET', $getEventsUrl)
          // Add the user's timezone to the Prefer header
          ->addHeaders(array(
            'Prefer' => 'outlook.timezone="'.$viewData['userTimeZone'].'"'
          ))
          ->setReturnType(Model\Event::class)
          ->execute();

        return response()->json($events);
      }

      private function getGraph(): Graph
      {
        // Get the access token from the cache
        $tokenCache = new TokenCache();
        $accessToken = $tokenCache->getAccessToken();

        // Create a Graph client
        $graph = new Graph();
        $graph->setAccessToken($accessToken);
        return $graph;
      }
    }
    ```

    考虑此代码执行的操作。

    - 将调用的 URL 为 `/v1.0/me/calendarView` 。
    - `startDateTime`和 `endDateTime` 参数定义视图的开始和结束。
    - `$select`参数将为每个事件返回的字段限制为仅显示视图实际使用的字段。
    - `$orderby`参数按其创建日期和时间对结果进行排序，最新项目最先开始。
    - `$top`参数将结果限制为25个事件。
    - `Prefer: outlook.timezone=""`标头会导致响应中的开始和结束时间调整为用户的首选时区。

1. 更新 **/routes/web.php** 中的路由，以将路由添加到此新控制器。

    ```php
    Route::get('/calendar', 'CalendarController@calendar');
    ```

1. 登录并单击导航栏中的 " **日历** " 链接。 如果一切正常，应在用户的日历上看到一个 JSON 转储的事件。

## <a name="display-the-results"></a>显示结果

现在，您可以添加一个视图，以对用户更友好的方式显示结果。

1. 在 **/resources/views** 目录中创建一个名为的新文件 `calendar.blade.php` ，并添加以下代码。

    :::code language="php" source="../demo/graph-tutorial/resources/views/calendar.blade.php" id="CalendarSnippet":::

    这将遍历一组事件并为每个事件添加一个表行。

1. `return response()->json($events);`从 `calendar` **/app/Http/Controllers/CalendarController.php**中删除操作的行，并将其替换为以下代码。

    ```php
    $viewData['events'] = $events;
    return view('calendar', $viewData);
    ```

1. 刷新页面，应用现在应呈现一个事件表。

    ![事件表的屏幕截图](./images/add-msgraph-01.png)
