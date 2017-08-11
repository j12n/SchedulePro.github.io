---
layout: post
title: "Awaiting Promises with ngClick"
date: 2017-08-11
---

At SchedulePro, we have a large JavaScript codebase built on [AngularJS](http://angularjs.org). While we've generally enjoyed constructing
our application with AngularJS, we've had a couple of cases where we've needed to deviate from standard practice. One of those that we've
recently altered is ngClick, the standard directive for executing code on a mouse click event.

ngClick has worked well for us for mutliple years, but we found that we kept repeating sections of code throughout various controllers
or html templates to prevent buttons from being double-clicked or supporting a disabled state during server processing. While the ideal
here would be to eliminate the delays from server processing, we've not yet prioritized the more dramatic shift throughout our UX.

Our initial approach at this was to implement a variable on the scope to track the disabled state of a button, commonly looking something
like:

<pre>
     &lt;span class="btn btn-default" ng-click="model.submit()" disabled="model.disabled"&gt;Submit&lt;/span&gt;
</pre>

And then within the backing JavaScript (simplified):

<pre>
    $scope.model.submit = function() {
        $scope.model.disabled = true;
        
        executePromise().then(function() {
            $scope.model.disabled = false;
        };
    }
</pre>

After polluting our code with this pattern in numerous places, we've finally tired of the exercise and looked for a new approach to the
common problem that we have been facing.
