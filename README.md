# Assets server

[![CircleCI build status](https://circleci.com/gh/canonical-web-and-design/assets.ubuntu.com.svg?style=shield)](https://circleci.com/gh/canonical-web-and-design/assets.ubuntu.com)
[![Coverage Status](https://coveralls.io/repos/github/ubuntudesign/assets-server/badge.svg?branch=main)](https://coveralls.io/github/ubuntudesign/assets-server?branch=main)

This is the codebase for [assets.ubuntu.com](https://assets.ubuntu.com), a Restful API service for storing and serving binary assets over HTTP, built with [Django REST framework](http://www.django-rest-framework.org/).

## Assets manager

We have also created [assets-manager](https://github.com/ubuntudesign/assets-manager/), a web interface for managing this assets server. It's hosted at [manager.assets.ubuntu.com](https://manager.assets.ubuntu.com).

## Local development

The simplest way to run the site locally is to first [install Docker](https://docs.docker.com/engine/installation/) (on Linux you may need to [add your user to the `docker` group](https://docs.docker.com/engine/installation/linux/linux-postinstall/)), and then use the `./run` script:

``` bash
./run
```

Once the containers are setup, you can visit <http://127.0.0.1:8017> in your browser.

### Creating tokens locally

To interact with the assets server, you'll need to generate an authentication token. You can do this for a locally running server with:

``` bash
$ ./run exec ./manage.py gettoken {token-name}
Token '{token-name}' created
3fe479a6b8184be4a4cdf42085f19f9a
```

You can list all tokens with [scripts/list-tokens.sh](scripts/list-tokens.sh) or delete them with [scripts/delete-token.sh](scripts/delete-token.sh).

See the API section below for how to use tokens.

### Tests

You can run all tests with:

``` bash
./run test
```

## Transforming images

Images can be transformed using the `op` get option, along with the required arguments.

You can manually specify one of these operations along with their corresponding options:
 - `region`
  - `rect`: The region as x,y,w,h; x,y: top-left position, w,h: width/height of region
 - `resize`
  - `w`: Width
  - `h`: Height
  - `max-width`
  - `max-height`
 - `rotate`
  - `deg`: Degrees to rotate the image

The default option is resize and can be used without setting `op`

```
https://assets.ubuntu.com/v1/4d7a830e-logo-ubuntuone.png?w=30
```

Or you can use another feature, like `region`:

```
https://assets.ubuntu.com/v1/4d7a830e-logo-ubuntuone.png?op=region&rect=0,0,50,50
```

## API functions

All API functions will need a token for access. Once you have one, you can then pass this token in the URL for requests to the assets server API. E.g.:

```
https://assets.ubuntu.com/v1/?token=3fe479a6b8184be4a4cdf42085f19f9a  # List all existing assets in JSON format
```

### Authentication tokens

You can see all tokens by simply going to "https://assets.ubuntu.com/v1/tokens?token=YOUREXISTINGTOKEN".

Anyone with a token can create another named token through the API using `curl`:

``` bash
$ curl --request POST --data name={new-token-name} "https://assets.ubuntu.com/v1/tokens?token={your-existing-token}"
{
    "message": "Token created", 
    "name": "{new-token-name}", 
    "token": "{the-new-generated-token}"
}
```

Even though at present all tokens have the same access, it's a good idea to generate a new token for each separate platform and each separate user of the API. This is so that if a token gets exposed, it can be easily deleted without impacting other clients.

#### Deleting tokens

Deleting tokens is similarly straightforward with `curl`:

``` bash
curl --request DELETE "https://assets.ubuntu.com/v1/tokens/{token-to-be-deleted}?token={your-api-token}"
```

### Uploading assets

You can upload assets with the cryptically named [`upload-assets`](https://github.com/canonical/canonicalwebteam.upload-assets) tool. This can be installed with `snap install upload-assets` or `sudo pip3 install upload-assets`.

It's usually best to store your API key in your RC file (e.g. `~/.bashrc`) by adding a line like `export UPLOAD_ASSETS_API_TOKEN=xxxxxx`. (You then probably need to `source ~/.bashrc` to load the environment variable).

You can then upload assets:

``` bash
$ upload-asset --api-domain localhost:8017 MY-IMAGE.png
{'url': u'http://localhost:8017/v1/xxxxx-MY-IMAGE.png', 'image': True, 'created': u'Tue Sep 27 16:13:22 2016', 'file_path': u'xxxxx-MY-IMAGE.png', 'tags': u''}
```

You can also use [curl](https://curl.haxx.se/docs/manpage.html):

``` bash
$ echo "asset=$(base64 -w 0 MY-IMAGE.png)" | \
  curl --request POST --data @- --data "friendly-name=MY-IMAGE.png" "https://assets.ubuntu.com/v1/?token={your-api-token}"
{
    "optimized": false,
    "created": "Wed Sep 28 11:07:33 2016",
    "file_path": "xxxxxxxx-MY-IMAGE.png",
    "tags": ""
}
```

### Deleting assets

**Warning: Please read this before deleting anything**

_The assets server serves all assets with a `cache-control` header instructing all clients to cache the asset for a year. This is to get the best possible performance on our websites. You need to bear this in mind before deleting any assets. If the asset has been cached by any clients - from our own Content Cache, to other intermediary caches, to users' browsers - you may struggle to get the assets deleted from those caches._

_For this reason it's best to find ways around needing to regularly delete assets._

_This is also why the [assets manager](https://manager.assets.ubuntu.com) doesn't support deleting assets through the interface._

To delete an assets, simply use `curl` with the `DELETE` method:

``` bash
curl --request DELETE "https://assets.ubuntu.com/v1/{asset-filename}?token={your-api-token}"
```

### Managing redirects

Since assets are cached for a very long time, if you know you will want to update the version of an assets behind a specific URL, this should be achieved by setting up a (non-permanent) redirect to the assets.

E.g., Every day we upload the latest Server Guide to e.g. `https://assets.ubuntu.com/v1/25868d7a-ubuntu-server-guide-2022-07-11.pdf`, and then we change the `https://assets.ubuntu.com/ubuntu-server-guide` URL to redirect to this latest version.

#### Creating redirects

You can set up a new redirect with `curl --data redirect_path={the-path-to-redirect} --data target_url={the-redirect-target} https://assets.ubuntu.com/v1/redirects?token={your-api-token}`.

E.g. this would create the `https://assets.ubuntu.com/ubuntu-server-guide` redirect mentioned above:

``` bash
$ curl --data redirect_path=ubuntu-server-guide --data target_url=https://assets.ubuntu.com/v1/25868d7a-ubuntu-server-guide-2022-07-11.pdf "https://assets.ubuntu.com/v1/redirects?token=xxxxxxxxxxx"
{
    "permanent": false, 
    "message": "Redirect created", 
    "target_url": "https://assets.ubuntu.com/v1/25868d7a-ubuntu-server-guide-2022-07-11.pdf", 
    "redirect_path": "ubuntu-server-guide"
}
```

#### Updating redirects

Once a redirect already exists, you can use the `PUT` method to update it using `curl --request PUT --data target_url={target-url} https://assets.ubuntu.com/v1/redirects/{redirect-path}?token={your-api-token}`.

E.g. (following the above example):

``` bash
$ curl --request PUT --data target_url=https://assets.ubuntu.com/v1/fe8d7514-ubuntu-server-guide-2022-07-13.pdf "https://assets.ubuntu.com/v1/redirects/ubuntu-server-guide?token=xxxxxxxxx"
{
    "target_url": "https://assets.ubuntu.com/v1/fe8d7514-ubuntu-server-guide-2022-07-13.pdf", 
    "permanent": false, 
    "redirect_path": "ubuntu-server-guide"
}
```

#### Deleting redirects

Deleting redirects is similarly simple, with `curl --request DELETE https://assets.ubuntu.com/v1/redirects/{redirect-path}?token={your-api-token}`, e.g. `curl --request DELETE https://assets.ubuntu.com/v1/redirects/ubuntu-server-guide?token=xxxxxxxxx`.

## Security

As the server only uses a basic token for authentication, it is paramount that in a production setting, the API functions are only accessed over HTTPS, to keep the API token secret. For this reason, when `DEBUG==false` the server will force a redirect to HTTPS for all API calls.

## Caching

The server is intended to be run in production behind a caching layer (e.g. [squid cache](http://www.squid-cache.org/)). And as the server stores assets by default with a unique hash corresponding to the file's contents (e.g. <code><b>a2f56da4</b>-some-image.png</code>), the cache expiration time should be as long as possible to maximise performance.
