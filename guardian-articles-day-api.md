# How to use The Guardian's API to download article data for content analysis (in Python 3.x)

The [Guardian offers an API](http://open-platform.theguardian.com/access/) as deep and robust as the[ New York Times Article API](http://developer.nytimes.com/docs/read/article_search_api_v2) when it comes to content analysis.

The Guardian's API offers more than ["1.7 million pieces of content", with published items as far back as 1999](http://open-platform.theguardian.com/access/). You can [register as a developer here](https://bonobo.capi.gutools.co.uk/register/developer), which gets you 5,000 API hits a day and an API key that looks something like this:

     26834484-d034-4638-b2ba-f4235af65dc2


The Guardian has a handy interactive explorer to [interactively tweak the query parameters](http://open-platform.theguardian.com/explore/).


## Search parameters

Here are the params for doing the broadest search for a day's content -- the `show-fields:all` key-pair will have the API return all available metadata, including the full text for articles when available.



|     key     |    value     |
|-------------|--------------|
| from-date   | 2016-03-04   |
| to-date     | 2016-03-04   |
| order-by    | newest       |
| show-fields | all          |
| page-size   | 200          |
| page        | 1            |
| api-key     | YOUR-API-KEY |

And this is the API endpoint:

      http://content.guardianapis.com/search

The full URL looks something like this:

http://content.guardianapis.com/search?from-date=2016-03-04&to-date=2016-03-04&order-by=newest&show-fields=all&page-size=200&api-key=YOUR_API_KEY_GOES_HERE



## Response meta JSON

The `page-size` param is maxed out at 200, so a script has to iterate through multiple pages per day (some days have nearly 500 items). Here's what the metadata of each response looks like, sans the `results` list:

~~~json
{
    "response": {
        "currentPage": 1,
        "orderBy": "newest",
        "pageSize": 200,
        "pages": 2,
        "results": ["..."],
        "startIndex": 1,
        "status": "ok",
        "total": 310,
        "userTier": "developer"
    }
}
~~~

## Article result JSON


Here's what a single result looks like -- I've truncated the `body` parameter as it contains the full HTML of the article:

~~~json
  {
    "fields": {
      "standfirst": "The IMF has changed its mind and realised Keynes's capital controls are a good thing. It's time to practise what they preach",
      "isPremoderated": "false",
      "lastModified": "2015-12-31T20:31:30.000Z",
      "liveBloggingNow": "false",
      "byline": "Kevin Gallagher",
      "commentable": "true",
      "hasStoryPackage": "false",
      "linkText": "Capital controls back in IMF toolkit | Kevin Gallagher",
      "commentCloseDate": "2010-03-04T23:50:00+00:00",
      "body": "<p>In 1942, when working to establish the International Monetary Fund, John Maynard Keynes said the &quot;control of capital movements, both inward and outward, should be a permanent feature of the post-war system.&quot;<br /> <br />In his new book <a href=\"http://press.princeton.edu/titles/9087.html\">Capital Ideas: The IMF and the Rise of Financial Liberalization</a>, Jeffrey Chwieroth argues that despite the fact that the economics profession largely maintained their support of Keynes&apos;s position, by the late 1990s the IMF motioned to change its articles of agreement in order to outlaw capital controls across the world.</p>",
      "shouldHideAdverts": "false",
      "headline": "Capital controls back in IMF toolkit",
      "legallySensitive": "false",
      "publication": "theguardian.com",
      "allowUgc": "false",
      "trailText": "<p><strong>Kevin Gallagher:</strong> The IMF has changed its mind and realised Keynes's capital controls are a good thing. It's time to practise what they preach</p>",
      "isInappropriateForSponsorship": "false",
      "shortUrl": "http://gu.com/p/2fayh",
      "wordcount": "745",
      "showInRelatedContent": "true",
      "productionOffice": "UK"
    },
    "webPublicationDate": "2010-03-01T23:50:33Z",
    "webTitle": "Capital controls back in IMF toolkit | Kevin Gallagher",
    "sectionName": "Opinion",
    "id": "commentisfree/cifamerica/2010/mar/01/imf-capital-controls",
    "sectionId": "commentisfree",
    "webUrl": "http://www.theguardian.com/commentisfree/cifamerica/2010/mar/01/imf-capital-controls",
    "apiUrl": "http://content.guardianapis.com/commentisfree/cifamerica/2010/mar/01/imf-capital-controls",
    "type": "article"
  }
~~~


## The Python script


Here's a quick Python script (specify `start_date` and `end_date`) to download the data in day-sized chunks into a local directory named `tempdata/articles`:

~~~py
import json
import requests
from os import makedirs
from os.path import join, exists
from datetime import date, timedelta

ARTICLES_DIR = join('tempdata', 'articles')
makedirs(ARTICLES_DIR, exist_ok=True)
# Sample URL
#
# http://content.guardianapis.com/search?from-date=2016-01-02&
# to-date=2016-01-02&order-by=newest&show-fields=all&page-size=200
# &api-key=your-api-key-goes-here

MY_API_KEY = open("creds_guardian.txt").read().strip()
API_ENDPOINT = 'http://content.guardianapis.com/search'
my_params = {
    'from-date': "",
    'to-date': "",
    'order-by': "newest",
    'show-fields': 'all',
    'page-size': 200,
    'api-key': MY_API_KEY
}


# day iteration from here:
# http://stackoverflow.com/questions/7274267/print-all-day-dates-between-two-dates
start_date = date(2012, 3, 1)
end_date = date(2012,4, 30)
dayrange = range((end_date - start_date).days + 1)
for daycount in dayrange:
    dt = start_date + timedelta(days=daycount)
    datestr = dt.strftime('%Y-%m-%d')
    fname = join(ARTICLES_DIR, datestr + '.json')
    if not exists(fname):
        # then let's download it
        print("Downloading", datestr)
        all_results = []
        my_params['from-date'] = datestr
        my_params['to-date'] = datestr
        current_page = 1
        total_pages = 1
        while current_page <= total_pages:
            print("...page", current_page)
            my_params['page'] = current_page
            resp = requests.get(API_ENDPOINT, my_params)
            data = resp.json()
            all_results.extend(data['response']['results'])
            # if there is more than one page
            current_page += 1
            total_pages = data['response']['pages']

        with open(fname, 'w') as f:
            print("Writing to", fname)

            # re-serialize it for pretty indentation
            f.write(json.dumps(all_results, indent=2))
~~~
