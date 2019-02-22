# Hello world add-on for Stremio in Python

## Adds a few public domain movies to Stremio

This example shows how to make a Stremio Add-on in Python with Rack.

## Quick Start

Clone or download this repo and run the following commands in the repo's root directory:

```sh
pip install flask
python3 stremio-addon.py
```

## Basic tutorial on how to re-create this add-on step by step

## Step 1: init a project

### Pre-requisites: git, pyton3, pip

This is the first, boilerplate step of creating an add-on for Stremio.

```sh
mkdir stremio-hello-world
cd stremio-hello-world
git init
git add *
git commit -a -m "initial commit"
```

## Step 2: Create `stremio-addon.py`

Let's start with our add-on basic functionality. Create `stremio-addon.py` with the following contents:

```python
from flask import Flask, Response, jsonify, url_for, abort
from functools import wraps

MANIFEST = {
    'id': 'org.stremio.helloworldPython',
    'version': '1.0.0',

    'name': 'Hello World Python Addon',
    'description': 'Sample addon made with Express providing a few public domain movies',

    'types': ['movie', 'series'],

    'catalogs': [],

    'resources': []
}

app = Flask(__name__)


def respond_with(data):
    resp = jsonify(data)
    resp.headers['Access-Control-Allow-Origin'] = '*'
    resp.headers['Access-Control-Allow-Headers'] = '*'
    return resp


@app.route('/manifest.json')
def addon_manifest():
    return respond_with(MANIFEST)

if __name__ == '__main__':
    app.run()
```

`respond_with` is a helper function that builds the response object and sets the required headers.

The manifest is the most important part of our add-on. It tells Stremio what are the add-on's capabilities. We define an `MANIFEST` object in the beginning of our scrip and the `Manifest` class serves it statically. The full specification of the add-on manifest is described [here](https://github.com/Stremio/stremio-addon-sdk/blob/master/docs/api/responses/manifest.md).

Now we can test that everything is working by running:

```shell
python3 stremio-addon.py
```

In the output you'll see a linke like this

> Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)

Now go to [http://127.0.0.1:5000/manifest.json](http://127.0.0.1:5000/manifest.json) and check if the add-on manifest is served correctly.

You can stop the add-on by pressing <kbd>Ctrl</kbd>+<kbd>C</kbd>.

## Step 3: Basic streaming

First let's update our manifest, so Stremio will know that our add-on support streams. Change the `resources` section of the manifest from:

```python
  'resources': []
```

to this:

```python
    'resources': [
        {'name': 'stream', 'types': [
            'movie', 'series'], 'idPrefixes': ['tt', 'hpy']}
    ]
```

To implement basic streaming, we will define a hash with a few public domain movies, just after the manifest definition.

```python
STREAMS = {
    'movie': {
        'tt0032138': [
            {'title': 'Torrent',
                'infoHash': '24c8802e2624e17d46cd555f364debd949f2c81e', 'fileIdx': 0}
        ],
        'tt0017136': [
            {'title': 'Torrent',
                'infoHash': 'dca926c0328bb54d209d82dc8a2f391617b47d7a', 'fileIdx': 1}
        ],
        'tt0051744': [
            {'title': 'Torrent', 'infoHash': '9f86563ce2ed86bbfedd5d3e9f4e55aedd660960'}
        ],
        'tt1254207': [
            {'title': 'HTTP URL', 'url': 'http://clips.vorwaerts-gmbh.de/big_buck_bunny.mp4'}
        ],
        'tt0031051': [
            {'title': 'YouTube', 'ytId': 'm3BKVSpP80s'}
        ],
        'tt0137523': [
            {'title': 'External URL',
                'externalUrl': 'https://www.netflix.com/watch/26004747'}
        ]
    },

    'series': {
        'tt1748166:1:1': [
            {'title': 'Torrent', 'infoHash': '07a9de9750158471c3302e4e95edb1107f980fa6'}
        ],

        'hpytt0147753:1:1': [
            {'title': 'YouTube', 'ytId': '5EQw5NYlbyE'}
        ],
        'hpytt0147753:1:2': [
            {'title': 'YouTube', 'ytId': 'ZzdBKcVzx9Y'}
        ]
    }
}
```

And then implement the `/stream/` path.

```python
@app.route('/stream/<type>/<id>.json')
def addon_stream(type, id):
    if type not in MANIFEST['types']:
        abort(404)

    streams = {'streams': []}
    if type in STREAMS and id in STREAMS[type]:
        streams['streams'] = STREAMS[type][id]
    return respond_with(streams)
```

**As you can see, this is an add-on that allows Stremio to stream 6 public domain movies and 3 series episode - in very few lines of code.**

Depending on your source, you can implement streaming (/stream/) or catalogs (/catalog/) of movie, series, channel or tv content types.

To load that add-on in the desktop Stremio, start the add-on, as described above, then click the add-on button (puzzle piece icon) on the top right, and write [http://127.0.0.1:9292/manifest.json](http://127.0.0.1:9292/manifest.json) in the "Addo-n Repository Url" field on the top left.

## Step 4: Implement catalog

We have 2 types of resources serving meta:

    /catalog/ serves basic metadata (id, type, name, poster) and handles loading of both the catalog feed and searching;

    /meta/ serves advanced metadata for individual items, for imdb ids though (what we're using in this example add-on), we usualy do not need to handle this method at all as it is handled automatically by Stremio's Cinemeta

**For now, we have the simple goal of loading the movies we provide on the top of Discover.**

Lets's tell Stremio that we will handle catalog.

We will update our manifest to reflect that.

```python
    'catalogs': [
            {'type': 'movie', 'id': 'Hello, Python'},
            {'type': 'series', 'id': 'Hello, Python'}
    ],

    'resources': [
        'catalog',
        {'name': 'stream', 'types': [
            'movie', 'series'], 'idPrefixes': ['tt', 'hpy']}
    ]
```

As you can see we define two catalogs - one for movies and one for series. In the `resources` section we state that the `catalog` resource is also supported by our add-on.

Now we can append our catalog data into the `stremio-addon.py` file:

```python
CATALOG = {
    'movie': [
        {'id': 'tt0032138', 'name': 'The Wizard of Oz', 'genres': [
            'Adventure', 'Family', 'Fantasy', 'Musical']},
        {'id': 'tt0017136', 'name': 'Metropolis',
         'genres': ['Drama', 'Sci-Fi']},
        {'id': 'tt0051744', 'name': 'House on Haunted Hill',
         'genres': ['Horror', 'Mystery']},
        {'id': 'tt1254207', 'name': 'Big Buck Bunny',
         'genres': ['Animation', 'Short', 'Comedy'], },
        {'id': 'tt0031051', 'name': 'The Arizona Kid',
         'genres': ['Music', 'War', 'Western']},
        {'id': 'tt0137523', 'name': 'Fight Club', 'genres': ['Drama']}
    ],
    'series': [
        {
            'id': 'tt1748166',
            'name': 'Pioneer One',
            'genres': ['Drama'],
            'videos': [
                    {'season': 1, 'episode': 1, 'id': 'tt1748166:1:1',
                        'title': 'Earthfall', 'released': '2010-06-16'}
            ]
        },
        {
            'id': 'hpytt0147753',
            'name': 'Captain Z-Ro',
            'description': 'From his secret laboratory, Captain Z-Ro and his associates use their time machine, the ZX-99, to learn from the past and plan for the future.',
            'releaseInfo': '1955-1956',
            'logo': 'https://fanart.tv/fanart/tv/70358/hdtvlogo/captain-z-ro-530995d5e979d.png',
            'imdbRating': 6.9,
            'genres': ['Sci-Fi'],
            'videos': [
                    {'season': 1, 'episode': 1, 'id': 'hpytt0147753:1:1',
                        'title': 'Christopher Columbus', 'released': '1955-12-18'},
                    {'season': 1, 'episode': 2, 'id': 'hpytt0147753:1:2',
                        'title': 'Daniel Boone', 'released': '1955-12-25'}
            ]
        }
    ]
}

# This is template we'll be using to construct URL for the item poster
METAHUB_URL = 'https://images.metahub.space/poster/medium/{}/img'
```

Then let's handle the catalog route.

```python
@app.route('/catalog/<type>/<id>.json')
def addon_catalog(type, id):
    if type not in MANIFEST['types']:
        abort(404)

    catalog = CATALOG[type] if type in CATALOG else []
    metaPreviews = {
        'metas': [
            {
                'id': item['id'],
                'type': type,
                'name': item['name'],
                'genres': item['genres'],
                'poster': METAHUB_URL.format(item['id'])
            } for item in catalog
        ]
    }
    return respond_with(metaPreviews)
```

If you run and install your add-on right now you'll notice that the new catalogs appear in the Stremio's Board and also you have this catalogs in the left pane of the discover. You can also play the videos from the sources we provide.

However, for the sake of completeness, we'll handle one more case, where Cinemeta is unable to find information about the content.

## Step 5: Implement meta

Maybe you have noticed that in the catalog that we defined in **Step 4**, there is more data than the one we provide on catalog request. There is also an item with `id` prefixed with `hpy` instead of `tt`.

Cinemeta looks up the media by IMDB ID so it can't find data for our `hpy` prefixed entry. This is the case where our add-on must provide that data.

We already have all the information needed, so let's expose it.

First we will again update our add-on's manifest.

```python
    'resources': [
        'catalog',
        # The meta call will be invoked only for series with ID starting with hpy
        {'name': "meta", 'types': ["series"], 'idPrefixes': ["hpy"]},
        {'name': 'stream', 'types': [
            'movie', 'series'], 'idPrefixes': ['tt', 'hpy']}
    ]
```

The `idPrefixes` defines a list of prefixes that we can handle. Stremio will not ask our add-on for data on `ID` that doesn't start with some of the predefined prefixes. If we omit this parameter, Stremio will ask our add-on for any item it encounters.

With the updated manifest, we are ready to handle the `/meta/` request.

```python
OPTIONAL_META = ["posterShape", "background", "logo", "videos", "description", "releaseInfo", "imdbRating", "director", "cast",
                 "dvdRelease", "released", "inTheaters", "certification", "runtime", "language", "country", "awards", "website", "isPeered"]


@app.route('/meta/<type>/<id>.json')
def addon_meta(type, id):
    if type not in MANIFEST['types']:
        abort(404)

    def mk_item(item):
        meta = dict((key, item[key])
                    for key in item.keys() if key in OPTIONAL_META)
        meta['id'] = item['id']
        meta['type'] = type
        meta['name'] = item['name']
        meta['genres'] = item['genres']
        meta['poster'] = METAHUB_URL.format(item['id'])
        return meta

    meta = {
        'meta': next((mk_item(item)
                      for item in CATALOG[type] if item['id'] == id),
                     None)
    }

    return respond_with(meta)
```

That's it. We are ready with our add-on. You can now load it into Stremio and start steaming.
