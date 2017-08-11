---
layout: post
title: "Awaiting Promises with ngClick"
date: 2017-08-11
author: Bo Rohlfsen
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

We had a couple of goals in mind when looking to resolve issues with ngClick:
<ol>
    <li>Adhere to DRY (Don't Repeat Yourself) principals within our codebase</li>
    <li>Prevent extra button clicks when awaiting server responses</li>
    <li>Don't break existing code flows</li>
    <li>Replace the default behavior, if possible, to eliminate competing directives</li>
</ol>

That in mind, we looked for existing solutions - some were quite novel - 
but none quite met our goals. A couple of sources hinted at leveraging a 
custom made directive alongside ngClick while others suggested an all-out
replacement for the directive, a la ngClickWait.

We weren't pleased with either of those paths, so we set out to create a
replacement for ngClick that would take the incoming action and wrap it in
a promise. Prior to execution, we'd update the disabled state of the 
control. Upon completion of the promise, we'd remove the disabled 
attribute.

This approach allowed us to accomplish our goals.
<ul>
    <li>We are able to now systematically go through and remove duplicate code, leaving just the ng-click attribute and purging the disabled attribute</li>
    <li>ng-click, by default, will now await all promises from the provided action</li>
    <li>For any existing code that had near-immediate execution, the simple wrapping as a promise created negligible change that is not observable to the user</li>
    <li>We were able to leave all of our existing ng-click usage as-is</li>
</ul>

Here is the directive we've created, along with the custom config to ensure that the default ngClick directive is removed to avoid conflicts.

<pre>
angular.module('ngClick.directive', [])

    .config(['$provide', function ($provide) {
        $provide.decorator('ngClickDirective', ['$delegate', function ($delegate) {
            // Drop the default ngClick directive
            $delegate.shift();
            return $delegate;
        }]);
    }])

    // Creation of a custom ngClick
    // Providing a method to the ng-click attribute will result in it's
    // full evaluation as a promise before re-enabling the element
    .directive('ngClick', ['$parse', '$q', function ($parse, $q) {

        return {
            restrict: 'A',
            priority: 0,
            compile: function($element, attr) {
                var fn = $parse(attr.ngClick);

                return function ngEventHandler(scope, element) {
                    element.on('click', function (event) {
                        var callback = function () {
                            $q.when(fn(scope, { $event: event })).finally(function () { 
                                element.attr('disabled', false);
                            });
                        };

                        element.attr('disabled', true);
                        scope.$apply(callback);
                    });
                };
            }
        };
    }]);
</pre>

We've just released this functionality into the wild this past week - so the verdict is still out on how we can continue to make great use of the feature. We'll keep you updated on how the refactoring goes from here forward.

-Cheers!