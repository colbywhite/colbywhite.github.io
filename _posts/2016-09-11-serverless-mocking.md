---
title: Serverless Mocking
---
# Be prepared to write Mocks in a serverless architecture

The amount of digital literature (aka blogs) that has been created detailing all of the various unit testing strategies 
you _have_ take while writing your project is overwhelming. Most of it is spot on and has extreme
value, but one engineer can only take so much preaching. So there's no need for me to add another sermon urging an
anonymous congregation to espouse their favorite programming language's preferred mocking framework.
 
But I have been hip-deep in a serverless environment and had an epiphany related to writing tests in such a world:
you're going to need to mock.

In this serverless terrain, you're likely to have functions that do nothing but talk to a third-party service. It is
generally frowned upon to have unit tests that depend on external services to be in a specific state. If you connect
those two dots, then you'll see that you're going to need a strategy to write tests without relying on The Internet.
And you will probably need it more so than a regular monolithic application. 

That's where mocks come in. Take the following quick example, a couple of small python utility methods that save JSON
content into AWS S3.

{% highlight python %}

BUCKET = 'blah'


def record_json(key, json_string, overwrite=False):
    if (not overwrite) and does_key_exist(key):
        print('%s already exists. Not overwriting.' % key)
    else:
        s3_client = boto3.resource('s3')
        s3_client.Bucket(BUCKET).put_object(
                ContentType='application/json', Key=key,
                Body=json_string
        )
        print('%s saved to S3' % key)


def does_key_exist(key):
    try:
        boto3.client('s3').head_object(Bucket=BUCKET, Key=key)
    except ClientError:
        return False
    else:
        return True
{% endhighlight %}


It's a pretty contrived example, but you can see that there is really only one statement that isn't S3-specific. That
`if (not overwrite) and does_key_exist(key)` statement is specific to the application and determines how exactly we 
want to interact with S3. There are likely going to several similar areas of code like this in a 
serverless application. 

We need to test our little sliver of application code without exercising the rest of the S3 code. First a small refactor
is in order to get the block that actually does the writing into it's own (poorly-named) function. 

```python
def record_json(key, json_string, overwrite=False):
    if (not overwrite) and does_key_exist(key):
        print('%s already exists. Not overwriting.' % key)
    else:
        _save_json_in_s3(key, json_string)

def _save_json_in_s3(key, json_string):
    s3_client = boto3.resource('s3')
    s3_client.Bucket(BUCKET).put_object(
            ContentType='application/json', Key=key,
            Body=json_string
    )
    print('%s saved to S3' % key)
```

This is another one of those "best practice" talking points. Keeping your methods small and specific makes them 
easier to test, easier to read, easier to maintain, etc. But it's even more important in a serverless application 
simply because there will be more of these blocks scattered around the code.
 
When you've got these isolated, it is now a lot easier to mock them out for your tests. Once mocked, we can assert
our `if` statement behaves appropriately in the various states as we expect, thus making sure no one accidentally 
changes the statement's logic in the future.

```python
from unittest import TestCase
from mock import MagicMock

import s3utils


class S3UtilsTest(TestCase):
    def setUp(self):
        self.json_string = '{"hello":"world"}'
        self.key = 'helloworld.json'
        s3utils._save_json_in_s3 = MagicMock()
        s3utils.does_key_exist = MagicMock()

    def test_record_json_writes(self):
        s3utils.record_json(self.key, self.json_string, True)
        s3utils._save_json_in_s3.assert_called_once_with(self.key, self.json_string)
        s3utils.does_key_exist.assert_not_called()

    def test_record_json_does_not_write_when_exists(self):
        # re-mock this method so it returns True
        s3utils.does_key_exist = MagicMock(return_value=True)

        s3utils.record_json(self.key, self.json_string, False)
        s3utils._save_json_in_s3.assert_not_called()
        s3utils.does_key_exist.assert_called_once_with(self.key)

    def test_record_json_writes_when_nonexistent(self):
        # re-mock this method so it returns True
        s3utils.does_key_exist = MagicMock(return_value=False)

        s3utils.record_json(self.key, self.json_string, False)
        s3utils._save_json_in_s3.assert_called_once_with(self.key, self.json_string)
        s3utils.does_key_exist.assert_called_once_with(self.key)
```

(In a less trivial example, it will likely be something more complicated than a single if statement, but you get the 
idea.)

This mocking concept isn't new. Any quick google search can flood your browser with a multitude of passionate lectures
on the value of the mocking strategy. Many engineers, including myself, would put it in that infamously cliched category
of "best practice" for unit testing. 

But what's interesting to me is how important it is in this brave new serverless world that is slowly overtaking the 
landscape. Simply the amount of code that will interact with other services warrants you brushing up on the details of
whatever mocking framework makes sense for your application. It's part of the territory.
