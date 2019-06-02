---
layout: post
title:  "Automating Apartment Hunting in San Francisco"
date:   2019-05-28 19:50:59 -0400
categories: python
---

I've lived in San Francisco for a little over 3 years and have "recently" been looking for my own apartment. I say "recently" beacuse in reality I've obsessively watched apartment pricing and availability for the last few months. Typically this invoves browsing listings on Craigslist and sometimes Padmapper, Hotpads, and Zillow [0].

During this search I've had the added luxury of not needing to move quickly. I live in a great apartment and neighborhood with some awesome roommates on a month-to-month lease. Moving has been about finding my own place, and I've hoped that having the flexiblity of time will make it easier to find a deal...whatever that means today in San Francisco.

However navigating Craigslist quickly got onerous. I tried copying + pasting listings into a google sheet and flagging some for follow-up but found myself needing to dedicate time to searching for apartments, deciding which were worth pursuing, and reaching out to make an introduction. 

I recently discovered (python-craigslist)[https://github.com/juliomalegria/python-craigslist] and ventured that if I could programmatically access Craigslist data I could at least provide myself with a nice, passive way to view apartment listings, and at best accelerate my search and beat the tens of people who apply for each unit (all while gettting to work on a fun python project!)

# Requirements

To start off, I made a few assumptions based on my experience searching for apartments and things I wanted to bulid to make it easier:
- I would focus on using Craigslist as the source of listing data
- I want to live in some specific neighborhoods in San Francisco and should bound the search to apartments in those locations
- I'd really like some way to view the results at my desk or on mobile. I've used Discord as a notification engine for some apps in the past, and thought it would be fun to use it for this too [1]

# Building Things

First off a lot of credit goes to (this post on dataquest.io)[https://www.dataquest.io/blog/apartment-finding-slackbot/] which helped me think through the steps to get something like this up and running. That said, I decided to design my own approach and write my own code so this could also be a educational experience for me. For context, I've been working on learning Python and have realized that projects like this one (read: something I'll use) are great motivation. 

To start, I  outlined the following rough steps:
1. Get data from Craigslist
2. Get an image showing where the listing is on a map
3. Post the result to a Discord channel

# Apartment listings

The `python-craigslist` module is really great and does all of the work here. It's well-documented and, while I wish it included some additional details for an individual listing like laundry specifics, I'm happy with the result. Getting data from the API is straightforward:

```python
housing = CraigslistHousing(site='sfbay', area='sfc', category='apa',
							filters={'posted_today' : True, 'min_price': 2000, 'max_price': 3600, 'min_bedrooms': 1})

for result in housing.get_results(sort_by='newest', geotagged=True):
	print(result)
```

Unsurprisingly, each result is JSON:
```
{'id': '1234567890', 'repost_of': None, 'name': 'omg it's an apartment!, 'url': 'https://sfbay.craigslist.org/sfc/apa/d/omg-its-an-apartment/1234567890.html', 'datetime': '2019-01-01 00:00', 'price': '$2200', 'where': 'San Francisco, CA', 'has_image': True, 'has_map': True, 'geotag': (37.3, -121.9), 'bedrooms': '1', 'area': 'mission'}
```

I didn't like some of the names from the response so I made a `listings` dictionary to rename some fields and to post-process some of the data:
```python
listing = {}
listing['craigslist_id'] = result['id']
listing['craigslist_url'] = result['url']
listing['posted_on'] = result['datetime']
listing['description'] = result['name']
listing['price'] = int(result['price'][1:])	# price always has a leading '$' so we need to strip the leading character
listing['neighborhood'] = str.lower(result['where']) if result['where'] else '' # sometimes this is null
listing['num_bedrooms'] = result['bedrooms']
listing['sqft'] = result['area']
listing['latitude'] = result['geotag'][0]
listing['longitude'] = result['geotag'][1]
```

Notably, the listing data includes location data (lat/long) if the person who created the listing provided it. This is great! I usually browse listings using a map view and like seeing where an apartment is. At first I decided I could just click through to the listing URL to see a map along with the listing details, but thought there had to be a way to automate this too.

# Mapping the listings

I came across the (Mapbox)[https://www.mapbox.com] API documentation a few weeks ago and filed it away in my list of "products I'd like to play with someday". The timing was serendipitious as I realized I could use the location data from each apartment listing + Mapbox to generate a map, which makes it _much_ easier to see where an apartment is at a glance. Unfortunately the Craigslist map data isn't perfect or is sometimes missing, but this still seemed better than having to click on individual links. While learning about Mapbox I wasn't quite able to decipher the specifics of their free tier but the numbers mentioned on https://www.mapbox.com/pricing/ seemed much higher than I would need for this project.

Mapbox has a (python package)[https://github.com/mapbox/mapbox-sdk-py] that makes this straightforward to implement. After browsing the API docs a bit I noticed I could have just constructed a URL with the appropriate parameters and been done with it, but I ended up using the python package because of how easy it was to set up (and how as a result I didn't have to deal with URL character + formatting issues)

```python
import config
from mapbox import Static

def get_map(latitude,longitude):
	service = Static(access_token=config.MAPBOX_ACCESS_TOKEN)

	point_on_map = {
		'type' : 'Feature',
		'properties' : {'name' : 'point'},
		'geometry' : {
			'type' : 'Point',
			'coordinates' : [-122.4267, 37.7689]
		}
	}

	response = service.image('mapbox.streets', retina=True, features=point_on_map, lon=-122.4267, lat=37.7689, z=15)
	
	return response.url
```

This code does a few things:
* Initialize `mapbox` with my API key (I'm storing secrets in `config.py` to keep those separate from the code)
* Define a point on the map with a style and some coordinates
* Make the request to Mapbox specifying: the map type, that I want a high-quality "retina" image, the previously-defined point to place on the map, and some coordinates to bound the map

For example, here's a map of San Francisco generated using a longitude of `-122.4267`, latitude of `37.7689`, and zoom of `z=12`. After playing around with the zoom value I settled on `z=15` for individual listings.

![San Francisco Map](/assets/sf-example.png)

`response.url` is an image, and after testing this with a few listings everything looked good! The next step was to pipe the listing data to Discord to receive alerts.

# Posting to Discord

Discord makes it super easy to set up alerts using [webhooks](https://support.discordapp.com/hc/en-us/articles/228383668-Intro-to-Webhooks) and I was surprised at how customizable an individual notification can be. I found this (Discord webhooks guide)[https://birdie0.github.io/discord-webhooks-guide/] invaluable when designing the formatting for the notification.

Once I set up a webhook endpoint, I used (Postman)[https://www.getpostman.com/] to start testing and customizing colors, text formatting, etc. I usually default to `cURL` or `requests` for API testing but the ease of edit --> make request made this a great use case for Postman. 

In the end I settled on the following raw JSON for the webhook endpoint:

```json
notification_data = {
	"embeds" : [{
		"title" : "${{price}} | {{description}}",
		"url" : "{{craigslist_url}}",
		"fields" : [
			{
				"name" : "Bedrooms",
				"value" : "{{num_bedrooms}}",
				"inline" : true
			},
			{
				"name" : "Square Feet",
				"value" : "{{num_sqft}}",
				"inline" : true
			}
		],
		"thumbnail" : {
			"url" : "{{map_url}}"
		},
		"footer" : {
			"text" : "posted {{datetime}} | {{neighborhood}}",
			"icon_url" : "https://i.imgur.com/r8jnedb.png"
		},
		"color" : 14431557
	}]
}
```

And the result:

![Discord example](/assets/discord-notification-example.png)

On colors -- I thought it would be fun to change the color of the Discord notification to add an additional visual indicator when glancing through the list. I went with green/yellow/red for "good"/"stretch"/"💸😭" depending on where a given listing falls within my desired price range. I thought about further restricting the min and max prices when getting listing data but wanted to give myself a chance to see apartments _outside_ of my budget, just in case there's a dream apartment out there that I'd regret passing over. I added the specific values to `settings.py`, and used (SpyColor)[https://www.spycolor.com] to convert hex color values to the decimal values required by Discord's API. 

```python
# settings.py
green_max = 3000
yellow_max = 3300

# main.py
if listing['price'] <= settings.green_max:
	color = 2664261  # green
elif listing['price'] > settings.green_max and listing['price'] <= settings.yellow_max:
	color = 16761095 # yellow
elif listing['price'] > settings.yellow_max:
	color = 14431557 # red
```

(Yes those price ranges are a bit ridiculous, but it is San Francisco after all)

After checking for which color to use, sending the notification is as easy as using `requests.post()` with the webhook URL and notification data JSON.

# Duplicate detection, decision making, and logging

To avoid duplicates I decided to save each retrieved listing to a database with the Craigslist listing ID as a key. I've worked with `sqlite` before and created a `databases.py` helper script that has functions for inserting, retrieving, and updating records. When I retrieve listings I first check the listing ID against the database to see if there's a duplicate. If there is, the listing is skipped (with the assumption that it was previously processed). 

Next, while I'm _intersted_ in seeing apartments from across the city I do have some neighborhoods that I'd rather not move to for commute or other reasons. I built a rudimentary filter by specifying a `neighborhood_blacklist` in a new `settings.py` file [2].

This allowed me to be a bit more selective with my notifications and came with an extra benefit -- it became quickly apparent that filtering based on neighborhood is far from perfect since the details are provided by the person who created the listing. These data are often inaccurate (or flat out wrong). To hedge against this I decided on the following approach:
* Store each listing
* If the listing's provided neighborhood is in the blacklist, don't notify me to avoid the noise
* Otherwise, notify and also update the listing record in the database to indicate that a message was sent
* Periodically manually review the "un-notified" listings to see if my filtering is too aggressive and adjust to taste

The blacklisting logic looks like this:

```python
# settings.py
neighborhood_blacklist = ['inner richmond', 'outer richmond', 'richmond', 
				'seacliff', 'tenderloin', 'inner sunset', 'outer sunset', 'sunset', 
				'bayview', 'parkside']

# main.py
if any(x in listing['neighborhood'] for x in settings.neighborhood_blacklist):
	notify = False
else:
	notify = True
```

# Operationalizing

I ran the script a few times scoped to different price ranges and verified everything was working as expected. Just in case, I also added some light logging at this point to help with future debugging. The only thing remianing was to schedule the script to run automatically.

Separately from this project I've been working on setting up a container / VM selfhosting environment on my own hardware. This project was a great candidate to host on my (Proxmox)[https://www.proxmox.com/en/] setup. There were a few ways to automate this but in the end I settled for hosting the code in an LXC container along with a cron job to run it periodically [3]. 

I'll spare the specifics of running this in my lab, but at a high level this included:
* spinning up a new container on my Proxmox host
* `git pull`-ing the source code onto the container
* adding secrets to `config.py`
* ensuring any dependencies (python, requirements) were installed
* setting up a cron job to run the main script hourly, e.g.,:

```bash
# crontab -e
1 * * * * python3 /path/to/oikos.py
```

I found (crontab guru)[https://crontab.guru/] to be a helpful resource in figuring out how to schedule cron jobs, especially for people like me who are somewhat new to this. I could have alternatively deployed this to a DigitalOcean box or Heroku but this ended up being just as easy (and free!)

And that's it! I now get hourly, cross-platform notifications for new apartment listings around San Francisco in my price range, with some nice maps and color-coding to make it easy to filter things visually. I've also started tagging listings in Discord by reacting with an emoji if I want to follow up. This has already been a huge help in my search: it's easy to browse Discord for listings that I previously flagged for follow-up when I have some downtime. I don't have to deal with having 10s of browser tabs open when searching for apartments, and I've started building a database of listing data that I can hopefully leverage to have a better understanding of housing market trends (and what a really is a "deal").

![Discord notifications with reactions](/assets/discord-notifications-with-reactions.png)

# What's next 

The code for this project is available on (GitHub)[https://#] in case you find it useful. Python 3.6+ is required. I'd really appreciate any comments on the code and structure!

## Follow-ups
* Some Craigslist listings can be fake (and it's easy to tell which when reading the listing details). It might be worth adding some logic to either (a) detect likely fake posts or (b) adding support for a "blacklist" feature in Discord that lets me mark a listing as fake.
* Many of the concepts here are extensible. I mentioned that I've used Discord for notifications before but in the past it's usually been with some software that has a "Discord notification" setting pre-defined. If anything I'm even more bullish on using Discord for custom app + service notifications after learning more about the webhook customization options 

# Notes

[0] I can pretty confidently say after months of browsing apartment data that virtually all listings available on other services are posted to Craigslist. This made me feel pretty good about defaulting to this versus other providers / APIs.

[1] I evaluated a few different notification services prior to deciding on Discord and that process is likely worth its own discussion. In the end I chose Discord for two reasons: unlimited integrations (unlike Slack) and a highly customizable, very simple webhook API.

[2] I did this to ensure secrets remained isolated in `config.py` while also creating a single place to edit variables I might want to modify later. I'm _not_ sure if this is the best or recommended approach for settings + configuration management but it worked for me.

[3] A post on my homelab setup is coming soon!