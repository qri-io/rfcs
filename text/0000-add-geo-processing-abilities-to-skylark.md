- Feature Name: Add geo processing abilities to starlark 
- Start Date: 2018-10-02
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary


# Motivation
[motivation]: #motivation

A lot of open public data, environmental data, government data etc has some kind of geospatial properties. Being able to 
perform geospatial operations on these datasets would be a huge win for qri and could help increase adoption of qri as a means
to distribute these types of dataset

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When working with spatial datasets, those associated with a point, line, polygon etc we often need to be able to perform manipulations
of the data based on those geometries. The Geo module of Starlib provides the basic data types and methods to be able to do just this.
Based on the python [Shapley](https://pypi.org/project/Shapely/) library the Geo module allows you to perform operations such as 

- Load Points Lines and Polygons in GeoJSON format and represent these as types within qri
- Calculate distances between geometric features 
- Perform filtering based on geometric conditions such as Within, Intersects, Contains etc 
- Manipulate lines and polygons with operations like intersects, union etc.

Some use-cases where this would be useful would be 

- Working with 311 complaint data from a open government data portal and wanting to produce a summary of counts of complaint by type in each census tract. This would require grabbing data from the dataset but also the census tracts for an area and performing a spatial join and aggregation on that data.
- Augmenting a dataset of bird sightings with the distance to the nearest wetlands.
- Calculating the overlap of flood regions and building footprints within a region to assess risk

An example of this which takes a dataset of 311 complaints, downloads the New York borough boundaries form the data portal and then counts how many complaints are in each borough, might look like this:

```python 
load("qri.sky", "qri")
load("http.sky", "http")
load("geo.sky", "geo")

def download(ds):

	# Download the New York Borough Boundaries
	res = http.get("http://data.beta.nyc//dataset/68c0332f-c3bb-4a78-a0c1-32af515892d6/resource/7c164faa-4458-4ff2-9ef0-09db00b509ef/download/42c737fd496f4d6683bba25fb0e86e1dnycboroughboundaries.geojson"
	
	boroughs = res.json()
	ds.set_body({complaints: ds, boroughs: boroughs})
	
	return 
	
def transform(qri):
   body = qri.get_body()
   complaints = body['complaints']
   boroughs = body['boroughs']
   
   # Parse then to GeoJSON ( returns an array of each geometry and a dictionary of  properties on that GeoJSON object) 
	boundaries, properties = geo.parseGeoJSON(boroughs)
	
	boro_names = [ boro['name'] for boro in properties]
	borough_counts = dict(zip(boro_names, [0]*len(boro_names))
	
	# Count the amount in each borough
	for complaint in complaints:
		point = geo.Point(complaint['latitude'], complaint['longitude'])
		
		for boro_name, geom in zip(boro_names, boundaries):
			if geo.within(point, geom):
				boro_counts[boro_name] += 1
	
	ds.set_body(boro_counts)

    
```


<!-- Explain the proposal as if it was already included in the language and you were teaching it to a Qri _developer_. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Qri developer should *think* about the feature, and how it should impact the way they use Qri. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to a Qri developer vs a Qri _user_.

For implementation-oriented RFCs (e.g. for Qri codebase internals), this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Types 

### Point 

Point constructor takes an x(longitude) and y(latitude) value and returns a Point object

```python 
load("geo.sky", "geo")

p = geo.Point(-44.34, 33)
print(p.x,p.y)
print(p.lat,p.lng)
```
### Line 
Takes either an array of coordinate pairs or an array of point objects and returns the line 
that connects them.

```python
load("geo.sky", "geo")

line = geo.Polygon( [[
        -120.58593749999999,
        58.44773280389084
      ],
      [
        -118.47656249999999,
        50.51342652633956
      ],
      [
        -113.203125,
        55.178867663281984
      ]])
line2 = geo.Polygon( [ geo.Point([-44.34 , 33.2]), geo.Point([-44.0,33.6]), geo.Point([-44.22, 33.5])])
```


### Polygon 
Takes an array of arrays of coordinate pairs ( or point objects) that define the outer boundary
and any holes / inner boundaries that represent a polygon. In GIS tradition, arrays of coordinates
that wind clockwise are filled regions and anti-clockwise represent holes.


```python 
load("geo.sky", "geo")
    
p = geo.Polygon( [
        # Outer boundary
        [[
          -93.515625,
          54.16243396806779
        ],
        [
          -99.49218749999999,
          42.5530802889558
        ],
        [
          -72.0703125,
          32.24997445586331
        ],
        [
          -72.0703125,
          43.83452678223682
        ],
        [
          -72.0703125,
          54.36775852406841
        ],
        [
          -80.85937499999999,
          57.326521225217064
        ],
        [
          -93.515625,
          54.16243396806779
        ]
        ]],

        # Hole in Polygon
        [[
          -87.1875,
          49.61070993807422
        ],
        [
          -87.890625,
          44.59046718130883
        ],
        [
          -81.2109375,
          43.83452678223682
        ],
        [
          -80.5078125,
          48.22467264956519
        ],
        [
          -87.1875,
          49.61070993807422
        ]]
])
```

## Geographic operations 

### Point based 

#### Buffer
Generates a buffered region of x units around a point.

```python
load("geo.sky", "geo")
   
p = geo.Point(-44.34,33)
polygon = p.buffer(200)
```

#### Distances 

```python 
load("geo.sky", "geo")
   
p1 = geo.Point(-44.34,33)
p2 = geo.Point(-44.34,32)

# Euclidian Distance
p1.distance(p1,p2) 

# Distance on the surface of a sphere with the same radius as Earth
p1.distanceGeodesic(p1,p2) 
```

### K-Nearest 

Given a target point T and an array of other points, return the K nearest points to T.

```python
load("geo.sky", "geo")
   
target = geo.Point(-44.34,33)
testPoints = [
	geo.Point(-44.34,32),
	geo.Point(-44.36,33.1),
	geo.Point(-46.2,32.13),
	geo.Point(-45.34,32.1),
	geo.Point(-46.12,32.1)
]

closest = target.KNN(testPoints, k = 3)
```


### Great Circle
Returns the great circle line segment between point 1 and 2.

```python
load("geo.sky", "geo")

p1 = geo.Point(-44.34,33)
p2 = geo.Point(-44.34,32)
line = p1.greatCircle(p2)
```

## Line operations

### Buffer

Given a line segment, return the polygon that represents that line segment buffered by X units.

```python 
load("geo.sky", "geo")
   
line  =  geo.Line([
	[
    	-120.58593749999999,
       58.44773280389084
    ],
    [
       -118.47656249999999,
       50.51342652633956
     ],
     [
        -113.203125,
        55.178867663281984
     ]
 ])
        
poly  = line.buffer(20)
```

## Length 

Given a line segment, calculate its length.

```python 
load("geo.sky", "geo")

line = geo.Line([
  [
    -120.58593749999999,
    58.44773280389084
  ],
  [
    -118.47656249999999,
    50.51342652633956
  ],
  [
    -113.203125,
    55.178867663281984
  ]
])
   line.length()
   line.geodesicLength()
```



## Geographic joins

Geographic joins allow us to do boolean tests with geometries. There are a potentially more of these we might 
want to implement but for the first pass Within and Intersects are the main ones we will focus on.

### Within 

Returns True if geometry A is entirely contained by geometry B

```python
load("geo.sky", "geo")
   
l = geo.Line([
      [
        -120.58593749999999,
        58.44773280389084
      ],
      [
        -118.47656249999999,
        50.51342652633956
      ],
      [
        -113.203125,
        55.178867663281984
      ]
    ])
p1 = geo.Point([
      -65.390625,
      48.45835188280866
    ])
poly1 = geo.Polygon([
      [
        [
          -93.515625,
          54.16243396806779
        ],
        [
          -99.49218749999999,
          42.5530802889558
        ],
        [
          -72.0703125,
          32.24997445586331
        ],
        [
          -72.0703125,
          43.83452678223682
        ],
        [
          -72.0703125,
          54.36775852406841
        ],
        [
          -80.85937499999999,
          57.326521225217064
        ],
        [
          -93.515625,
          54.16243396806779
        ]
      ]
 ])

poly2 = geo.Polygon([
      [
        [
          -93.515625,
          54.16243396806779
        ],
        [
          -99.49218749999999,
          42.5530802889558
        ],
        [
          -72.0703125,
          32.24997445586331
        ],
        [
          -72.0703125,
          43.83452678223682
        ],
        [
          -72.0703125,
          54.36775852406841
        ],
        [
          -80.85937499999999,
          57.326521225217064
        ],
        [
          -93.515625,
          54.16243396806779
        ]
      ]
  ])

geo.within(p1, poly1) # False 
geo.within(poly2,poly1) # True 
geo.within(line, poly2) # False
```


### Intersects 

Similar to within but part of geometry B can lie outside of geometry A and it will still return True

```python
load("geo.sky", "geo")
    
geo.intersects(poly1,poly2) # True 
geo.intersects(line,poly1) # False
```

## GeoJSON

One of the things the library will need to do is to be able to parse common formats into 
geometries. As qri currently supports JSON I propose we focus on working with GeoJSON for now 
and we can add support for other common spatial data types later.

```python 
load("geo.sky", "geo")

geoms, properties = geo.parseGeoJSON({
	{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {
        ufo_sightings: 20,
        population: 40
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          -103.0078125,
          38.272688535980976
        ]
      }
    },
    {
      "type": "Feature",
      "properties": {
        ufo_sightings: 100,
        population: 2
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          -87.1875,
          47.989921667414194
        ]
      }
    }
  ]
}
})

# this would result in geoms being an array of geo.Point objects and properties being an array of the point properties [ {ufo_sightings: 20, population: 40}, {ufo_sightings: 100, population: 2}
 
```


<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?
This would only really be relevant for geospatial datasets. While this is something that is probably going
to be important for environmental, urban and government datasets if these are not an important focus for 
qri the effort here might not be warranted.  

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is a subset of functionality that has been battle tested in industry standard solutions like 
PostGIS. The API is following that of the Shapley python library so should be familiar to most geospatial 
python developers.
<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->



# Prior art
[prior-art]: #prior-art

- [PostGIS Functions](https://postgis.net/docs/reference.html#Geometry_Editors) : A breakdown of geospatial functions that PostGIS uses (this is probably the most comprehensive list of these kinds of functions)
- [golang-geo](https://github.com/kellydunn/golang-geo) : A geospatial library for Go
- [https://github.com/golang/geo ](https://github.com/golang/geo) : Googles S2 library implement in Go
- [Shapley](https://pypi.org/project/Shapely/) : One of the core python libraries for geometries, main inspiration for the API here.
- [GeoPandas](http://geopandas.org/) : DataFrame implementation that has geospatial functionality.

<!-- Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other places and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other projects, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that Qri sometimes intentionally diverges from other projects. -->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

There are a bunch of features that this proposal doesn't include, is it valid to not include these in the first pass:
- Projections : Geospatial data can be projected in to many different local coordinate systems which are 
useful in different contexts. We aren't including this in this first implementation but if it's important we could include it
- Geometry modification methods : Translate, Rotate etc functions 

Other questions:

- Do we want this to help support the visualization part of Qri when data is geospatial?
- What volumes of data should this be looking to support? Do we need to be building and somehow persisting with the dataset geospatial indexes that allow for faster processing / querying? 
- Support for other geospatial data formats : Qri currently supports csv, json and cbor formats. JSON can be used to work with GeoJSON but other geospatial formats like KML, Geopackage and  Shape Files to name a few are common. Would these potentially be supported by the cbor format? If not what is the scope for adding formats like these?

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
