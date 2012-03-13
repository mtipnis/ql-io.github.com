---
title: Build Your Own APIs
layout: post
author: cpenkar
---

A route in ql.io is a new consumer-optimized HTTP interface. Routes superimpose a simple and familiar HTTP interface on ql.io scripts without needing to specify an elaborate script in the request. In other words, routes make ql.io a platform to "build your own APIs". 
  
Having built this capability, in this post I want to highlight some potential ways to take advantage of routes.

<!-- more -->

## Discovery

Let's imagine you built some new APIs using ql.io. How can your users find more about those APIs? Here is how.

Every ql.io now includes a special URI `http://{host}:{port}/api` that lets you browse through the following:

* List of routes
* For each route, a URI template (or a URI), and HTTP method supported.
* For each route, the tables used
* For each table, the original HTTP resource that it maps to.

Here is an example.

![API browsing](/images/2012-03-12-byoa-1.png)

Go to [http://ql.io/apis](http://ql.io/apis) to try it out. Though this capability is automatic, it needs a couple of actions as you build tables and routes.

## Parametrizing

Let us consider the following script.

    keyword = "ql.io";
    web = select web:Title, web:Url, web:Description from bing.search where q = "{keyword}";
    tweets = select id as id, from_user_name as user_name, text as text
        from twitter.search where q = "{keyword}";
 
    return {
      "keyword": "{keyword}",
      "web": "{web}",
      "tweets": "{tweets}"
    }
	
Since `keywords` is hardcoded in this script, it is only capable of finding "ql.io" from twitter and bing. You can parameterize this script by defining it as a route.

    web = select web:Title, web:Url, web:Description from bing.search where q = "{keyword}";
    tweets = select id as id, from_user_name as user_name, text as text
        from twitter.search where q = "{keyword}";
 
    return {
      "keyword": "{keyword}",
      "web": "{web}",
      "tweets": "{tweets}"
    } via route '/search?q={keyword}' using method get;

With this route, you can use `http://{host}:{port}/search?q={your keyword here}` to run the script. Try this route at [http://ql.io/search?q=ql.io](http://ql.io/search?q=ql.io).

## Similar but Different

There are other ways to parameterize routes. Let's say, you would like to provide the following resources to client apps.

1. `http://{host}:{port}/item/location?itemid={itemid}` with method `GET`: Retrieve geo-location for a given item id.
2. `http://{host}:{port}/item/location?keyword={keyword}` with method `GET`: Given a keyword, find matching items, and then find and return their geo-locations.
3. `http://{host}:{port}/item/location?itemid={itemid}&keyword={keyword}` with method `GET`: Given a keyword and item ID, find geolocatons of all items matching the keyword, and also for the item ID. Such a response may be useful when you want to show locations of a given items, but also other items that match the keyword. 

Notice all the routes above have the same HTTP method `GET` and the same path `/item/location` but differ in query parameters. But as the aggregation logic may be different for each of these scripts, you can define three different scripts for the same path.

Here is a route for `/item/location?itemid={itemid}`
 
    -- Matches request  /item/location?itemid=140716431558 
    --
	return select e.ItemID as id, e.Title as title, e.ViewItemURLForNaturalSearch as url, 
                  g.geometry.location as latlng
                  from details  as e, google.geocode as g
  	               where e.itemId = "{itemid}"
                         and g.address = e.Location
           via route '/item/location?itemid={itemid}' using method get;
   
Here is the route for `/item/location?keyword={keyword}`.

    -- Matches request  /item/location?keyword=ipad 
    --
	return select e.ItemID as id, e.Title as title, e.ViewItemURLForNaturalSearch as url, 
                 g.geometry.location as latlng
                 from details  as e, google.geocode as g
                 where e.itemId in (select itemId from finditems where keywords = "{keyword}")
                       and g.address = e.Location
           via route '/item/location?keyword={keyword}' using method get;
        
Finally, here is the route for `/item/location?itemid={itemid}&keyword={keyword}`

    -- Matches request  /item/location?keyword=ipad&itemid=140716431558 
    --
	keywordResult =  select e.ItemID as id, e.Title as title, e.ViewItemURLForNaturalSearch as url, 
                            g.geometry.location as latlng
                            from details  as e, google.geocode as g
  	                        where e.itemId in (select itemId from finditems where keywords = "{keyword}")
                                  and g.address = e.Location
	itemidResult =  select e.ItemID as id, e.Title as title, e.ViewItemURLForNaturalSearch as url, 
                           g.geometry.location as latlng
  	                       from details  as e, google.geocode as g
  	                       where e.itemId = "{itemid}"
                                 and g.address = e.Location
    return {
	    "keywordResult": "{keywordResult}",
        "itemidResult" : "{itemidResult}"
    } via route '/item/location?itemid={itemid}&keyword={keyword}' using get;

Given a request URI, ql.io's routing engine is capable of matching the request to one of these scripts.

## Non-Idempotent and Unsafe

Route parameterization is not limited to HTTP `GET` alone. You can allow clients to supply bodies with `POST`, `PUT`, `PATCH` and `DELETE` requests.

Consider the following table for bitly APIs.

    create table bitly.shorten
      on insert get from "http://api.bitly.com/v3/shorten?login={^login}&apiKey={^apikey}&longUrl={^longUrl}&format={format}"
                using defaults apikey = "{config.bitly.apikey}", login = "{config.bitly.login}", format = "json"
                using patch 'bitly.js'
                resultset 'data.url'
      on select get from "http://api.bitly.com/v3/expand?login={^login}&apiKey={^apikey}&shortUrl={^shortUrl}&format={format}"
                using defaults apikey = "{config.bitly.apikey}", login = "{config.bitly.login}", format = "json"
                using patch 'bitly.js'
                resultset 'data.expand'

To shorten a URI, you may want to add a route that uses HTTP method `POST`.

    return insert into bitly.shorten (longUrl) values ('{uri}')
        via route '/bitly/shorten' using method post;

You can then use one of the following URIs to shorten a URI.

    curl http://localhost:3000/bitly/shorten -X POST 
      -H 'Content-Type: application/json' 
      -d '{"uri": "http://www.ebay.com"}'

    curl http://localhost:3000/bitly/shorten -X POST 
      -H 'Content-Type: application/xml' 
      -d '<uri>http://www.ebay.com</uri>'

    curl http://localhost:3000/bitly/shorten -X POST 
      -H 'Content-Type: application/x-www-form-urlencoded' 
      -d 'uri=http://www.ebay.com'

In each case, ql.io's routing engine will coerce the body into an associative array, and substitutes the values for tokens in the routing script or tables used by the routing script.

## Markdown

ql.io scripts can include [markdown](http://daringfireball.net/projects/markdown/) based line comments that begin with `--`. ql.io treats comments preceding `return` statements in routing scripts, and comments preceding `create table` statements as documentation.

    -- Use this resource to shorten a URI using bitly.

    return insert into bitly.shorten (longUrl) values ('{uri}')
        via route '/bitly/shorten' using method post;

We've some work to do to support [multi-line comments](https://github.com/ql-io/ql.io/issues/340), but you get the idea!