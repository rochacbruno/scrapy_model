Create scraper using Scrapy Selectors
============================================

[![Build
Status](https://travis-ci.org/rochacbruno/scrapy_model.png)](https://travis-ci.org/rochacbruno/scrapy_model)

[![PyPi version](https://pypip.in/v/scrapy_model/badge.png)](https://pypi.python.org/pypi/scrapy_model/)
[![PyPi downloads](https://pypip.in/d/scrapy_model/badge.png)](https://pypi.python.org/pypi/scrapy_model/)

## What is Scrapy?

Scrapy is a fast high-level screen scraping and web crawling framework, used to crawl websites and extract structured data from their pages. It can be used for a wide range of purposes, from data mining to monitoring and automated testing.

http://scrapy.org/


## What is scrapy_model ?

It is just a helper to create scrapers using the Scrapy Selectors allowing you to select elements by CSS or by XPATH and structuring your scraper via Models (just like an ORM model) and plugable to an ORM model via ``populate`` method. 

Import the BaseFetcherModel, CSSField or XPathField (you can use both)

```python
from scrapy_model import BaseFetcherModel, CSSField
```

Go to a webpage you want to scrap and use chrome dev tools or firebug to figure out the css paths then considering you want to get the following fragment from some page.

```html
    <span id="person">Bruno Rocha <a href="http://brunorocha.org">website</a></span>
``` 

```python
class MyFetcher(BaseFetcherModel):
    name = CSSField('span#person')
    website = CSSField('span#person a')
    # XPathField('//xpath_selector_here')
```

Every method named ``parse_<field>`` will run after all the fields are fetched for each field.

```python
    def parse_name(self, selector):
        # here selector is the scrapy selector for 'span#person'
        name = selector.css('::text').extract()
        return name
        
    def parse_website(self, selector):
        # here selector is the scrapy selector for 'span#person a'
        website_url = selector.css('::attr(href)').extract()
        return website_url

```


after defined need to run the scraper


```python

fetcher = Myfetcher(url='http://.....')  # optionally you can use cached_fetch=True to cache requests on redis
fetcher.parse()
```

Now you can iterate ``_data``, ``_raw_data`` and atributes in fetcher

```python
>>> fetcher.name
<CSSField - name - Bruno Rocha>
>>> fetcher.name.value
Bruno Rocha
>>> fetcher._data
{"name": "Bruno Rocha", "website": "http://brunorocha.org"}
```

You can populate some object

```python
>>> obj = MyObject()
>>> fetcher.populate(obj)  # fields optional

>>> obj.name
Bruno Rocha
```

If you do not want to define each field explicitly in the class, you can use a json file to automate the process

```python
class MyFetcher(BaseFetcherModel):
   """ will load from json """
   
fetcher = MyFetcher(url='http://.....')
fetcher.load_mappings_from_file('path/to/file.json')
fetcher.parse()
```

In that case file.json should be

```json
{
   "name": {"css", "span#person"},
   "website": {"css": "span#person a"}
}
```

You can use ``{"xpath": "..."}`` in case you prefer select by xpath


### Instalation

easy to install

If running ubuntu maybe you need to run:

```bash
sudo apt-get install python-scrapy
sudo apt-get install libffi-dev
sudo apt-get install python-dev
```

then

```bash
pip install scrapy_model
```

or


```bash
git clone https://github.com/rochacbruno/scrapy_model
cd scrapy_model
pip install -r requirements.txt
python setup.py install
python example.py
```

Example code to fetch the url http://en.m.wikipedia.org/wiki/Guido_van_Rossum

```python
#coding: utf-8

from scrapy_model import BaseFetcherModel, CSSField, XPathField


class TestFetcher(BaseFetcherModel):
    photo_url = XPathField('//*[@id="content"]/div[1]/table/tr[2]/td/a')

    nationality = CSSField(
        '#content > div:nth-child(1) > table > tr:nth-child(4) > td > a',
    )

    links = CSSField(
        '#content > div:nth-child(11) > ul > li > a.external::attr(href)',
        auto_extract=True
    )

    def parse_photo_url(self, selector):
        return "http://en.m.wikipedia.org/{}".format(
            selector.xpath("@href").extract()[0]
        )

    def parse_nationality(self, selector):
        return selector.css("::text").extract()[0]

    def parse_name(self, selector):
        return selector.extract()[0]

    def post_parse(self):
        # executed after all parsers
        # you can load any data on to self._data
        # access self._data and self._fields for current data
        # self.selector contains original page
        # self.fetch() returns original html
        self._data.url = self.url


class DummyModel(object):
    """
    For tests only, it can be a model in your database ORM
    """


if __name__ == "__main__":
    from pprint import pprint

    fetcher = TestFetcher(cache_fetch=True)
    fetcher.url = "http://en.m.wikipedia.org/wiki/Guido_van_Rossum"

    # Mappings can be loaded from a json file
    # fetcher.load_mappings_from_file('path/to/file')
    fetcher.mappings['name'] = {
        "css": ("#section_0::text")
    }

    fetcher.parse()

    print "Fetcher holds the data"
    print fetcher._data.name
    print fetcher._data

    # How to populate an object
    print "Populating an object"
    dummy = DummyModel()

    fetcher.populate(dummy, fields=["name", "nationality"])
    # fields attr is optional
    print dummy.nationality
    pprint(dummy.__dict__)

```

# outputs


```
Fetcher holds the data
Guido van Rossum
{'links': [u'http://www.python.org/~guido/',
           u'http://neopythonic.blogspot.com/',
           u'http://www.artima.com/weblogs/index.jsp?blogger=guido',
           u'http://python-history.blogspot.com/',
           u'http://www.python.org/doc/essays/cp4e.html',
           u'http://www.twit.tv/floss11',
           u'http://www.computerworld.com.au/index.php/id;66665771',
           u'http://www.stanford.edu/class/ee380/Abstracts/081105.html',
           u'http://stanford-online.stanford.edu/courses/ee380/081105-ee380-300.asx'],
 'name': u'Guido van Rossum',
 'nationality': u'Dutch',
 'photo_url': 'http://en.m.wikipedia.org//wiki/File:Guido_van_Rossum_OSCON_2006.jpg',
 'url': 'http://en.m.wikipedia.org/wiki/Guido_van_Rossum'}
Populating an object
Dutch
{'name': u'Guido van Rossum', 'nationality': u'Dutch'}
```
