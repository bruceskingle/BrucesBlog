# API Versioning
Or, "Why do we need pesky version numbers in REST API Endpoints, and why do they keep getting _deprecated_?"

If the world stood still, then everything would be easier (if a little less exciting). As things are, most online service providers who provide REST APIs work hard to develop and deliver new functionality to their customers, and [Symphony](http://www.symphony.com), where I work, is no exception. Obviously it would be nice if the design of everything was "right first time" and everything we might ever want to provide in an interface was designed "up front", and we do our best to do this as far as we can. The fact is, however, that things have a habit of "cropping up later" and changes inevitably become necessary. The question then arises of managing that change, and this is where the pesky version numbers come in.

As an example, consider a User Search API, with a REST endpoint to find users in a database. The API has a couple of methods:

```
GET /users
```

This method returns a list of all users in the system.


```
GET /users?email=someone@some.domain
```

This method returns a single user whose email address is given, or returns an HTTP 404 response if there is none.

So far, so good.

## Customer #1
Now, the first customer of this service is **Mom n' Pop, LLC** who are using the service because their business has grown to the point where they can't manage it on spreadsheets any more, they have 35 employees, whose details they add to the system.

Every week Mom n' Pop run a process to pay their staff, they call **GET /users** and iterate through the list of employees calculating each person's pay, and all is well. During the development of the payroll application, Mom n' Pop's teenage son (who does the computer stuff for Mom n' Pop) wonders if he needs to ensure that he gets all the employees and tries calling 

```
GET /users?maxRows=200
```

Which seems to work, but is that parameter actually doing anything? He tries calling

```
GET /users?maxRows=20
```

The call works, but he still gets all 35 records back. He means to remove the useless parameter but in the rush to finish he forgets and the payroll continues to run satisfactorily for many weeks.

## Customer #2
The second customer of the service is **Big Company Plc** who have 223,000 employees in 20 countries. In the first week of Big Company's use of the service, there are a couple of problems.

The first one is that when the **GET /users** endpoint is called the server queries the database for all 223,000 users and constructs a JSON response containing them all. Unfortunately this consumes massive resources on the server, which crashes, bringing down the service for everyone. Mom n Pop fail to pay their payroll for the first time ever, and they are _not_ happy.

The second one is that users at Big Company sometimes share email addresses so the **GET /users?email=someone@some.domain** endpoint now sometimes returns more than one user, which breaks its contract, and the application calling it.

## What to Do?
The service provider needs to act quickly and they decide to add a limit to the fetch users API, which becomes

```
GET /users?maxRows=N
```

Since a very large query breaks the server they decide to put a default value of 32 on **maxRows**. They know that 32 is a nice round (if you are a computer nerd) number and it's bigger than the number of employees of their second largest customer, so it won't break their applications.

Since the query by email needs to be able to return more than one record, they have a choice, one idea is to change the method to return a JSON array instead of a JSON object, and to return 0, 1 or more records in that array.

Another approach is to return an array but to continue to return a 404 status code if there are no matches, but this seems a bit untidy.

A third approach is to keep the method the same, to return a 500 status code if there are multiple results, and to add a new method which returns an array with 0, 1 or more elements.

A variation on all of these is to accept the **maxRows** parameter and behave differently depending on whether it is present and what its value is, so that's options 4, 5 and 6.

## Plan A - Just Change it
In this approach the existing endpoints simply change to have the new behavior. No customer has to change their code, everything will "just work". 

And it does. 

The first 20 of Mom n' Pop's employees get paid right on time. The other 15? Not so much.

One of Big Company's apps, which was unable to retrieve multiple users with the same email address now works perfectly, and they didn't have to change a thing!

Another app, which was calling **GET /users?email=someone@some.domain** to validate email addresses and checking for a 404 status, but ignoring the response body, failed in a new and exciting way, which involved sending thousands of emails to people who are no longer employees of Big Company, with news about the confidential take over which was about to be announced.

## Plan B - New Endpoints
In this approach the existing endpoints remain unchanged, except that the fetch all users endpoint has a check added so that if the number of users in the database is more than 1,000 it returns an HTTP 500 status code. New endpoints with different names are added to provide the required functionality:

```
GET /pagedUsers?page=N
```

This endpoint returns at most 50 users in a "page" and the caller must keep calling the method, incrementing the page number until it gets fewer than 50 results.

Now the app at Big Company which tries to list all users breaks with an HTTP 500 response, but they can change it to call the new method, which now works.

A new email query endpoint is added

```
GET /usersList?email=someone@some.domain
```

This endpoint returns a JSON array. Callers who switch to the new endpint are good, but any who don't notice that it has been added continue to run the risk of an unexpected result the first time a shared email address appears.

# The Least Worst Option
The objective of a versioning policy on the API is to ensure that calling code does not break, and that new features can be added and bugs can be fixed, as smoothly as possible. Considerations are

- What to do about unrecognized parameters
- Whether to allow the addition of new optional parameters
- Whether to allow changes to the response payload
- Whether to allow new HTTP response codes

In general, it's helpful to allow "non-breaking" changes to existing endpoints, but in some cases it is surprisingly hard to tell if a change is "non-breaking" in all cases, as I hope the example above shows. The safest approach, in terms of not breaking existing working programs, is to allow no changes whatsoever to existing endpoints and create new ones for every little feature. Of course this means that callers have to keep changing their code to get every new feature.

Once you have multiple versions of the same end point the question of on-going support for obsolete versions comes up. Ideally the provider would support all versions for all time, but the reality is that the code to implement them all has to be maintained and that takes time and effort. Since all organizations have finite resources, the work to maintain old versions means that there is less time to develop new features. So one customer who refuses to move off a very old version of an endpoint, may be causing new features to be delayed, which would benefit all customers, including the one who is too busy to change their code calling the old version.

The compromise solution is to have a deprecation policy, which commits to supporting old versions for some reasonable time, but allows for them to be retired in a reasonable time scale.

As with any other design choice, the API versioning policy is a compromise between different concerns, the Symphony REST API versioning policy is at [https://rest-api.symphony.com/docs/api-change-management](https://rest-api.symphony.com/docs/api-change-management). We hope we have got the right balance, but we are open to new ideas and approaches if you have any suggestions.

