---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.1'
      jupytext_version: 1.2.3
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# Clustering and Regionalization

```python
from pysal.explore.esda.moran import Moran
import pysal.lib.weights.set_operations as Wsets
from pysal.lib.weights import Queen, KNN
from booktools import choropleth
import seaborn
import pandas
import geopandas
import numpy
from sklearn.cluster import KMeans, AgglomerativeClustering
import matplotlib.pyplot as plt
```

## Introduction

The world's hardest questions are complex and multi-faceted.
Effective methods to learn from data should recognize this. Many questions
and challenges are inherently multidimensional; they are affected, shaped, and
defined by many different components all acting simultaneously. In statistical
terms, these processes are called *multivariate processes*, as opposed to 
*univariate processes*, where only a single variable acts at once.
Clustering is a fundamental method of geographical analysis that draws insights
from large, complex multivariate processes. It works by finding similarities among the
many dimensions in a multivariate process, condensing them down into a simpler representation
Thus, through clustering, a complex and difficult to understand process is recast into a simpler one
that even non-technical audiences can look at and understand. 

Often, clustering involves sorting observations into groups. For these groups to be more
meaningful than any single initial dimension, members of a group should be more
similar to one another than they are to members of a different group.
Each group is referred to as a *cluster* while the process of assigning
objects to groups is known as *clustering*. If done well, these clusters can be
characterized by their *profile*, a simple summary of what members of a group
are like in terms of the original multivariate process.

Since a good cluster is more
similar internally than it is to any other cluster, these cluster-level profiles
provide a convenient shorthand to describe the original complex multivariate process.
Observations in one group may have consistently high 
scores on some traits but low scores on others. 
The analyst only needs to look at the profile of a cluster in order to get a
good sense of what all the observations in that cluster are like, instead of
having to consider all of the complexities of the original multivariate process at once. 
Throughout data science, and particularly in geographic data science, 
clustering is widely used to provide insights on the
geographic structure of complex multivariate spatial data. 

In the context of explicitly spatial questions, a related concept, the *region*,
is also instrumental. A *region* is similar to a *cluster*, in the sense that
all members of a region have been grouped together, and the region should provide 
a shorthand for the original data. 
Further, for a region to be analytically useful, its members also should
display stronger similarity to each other than they do to the members of other regions. 
However, regions are more complex than clusters because they combine this
similarity in profile with additional information about the geography of their members.
In short, regions are like clusters (since they have a coherent profile), but they
also have a coherent geography&mdash;members of a region should also be
located near one another.

The process of creating regions is called regionalization.
A regionalization is a special kind of clustering where the objective is 
to group observations which are similar in their statistical attributes,
but also in their spatial location. In this sense, regionalization embeds the same
logic as standard clustering techniques, but also applies a series of
spatial and/or geographical constraints. Often, these
constraints relate to connectivity: two candidates can only be grouped together in the
same region if there exists a path from one member to another member
that never leaves the region. These paths often model the spatial relationships
in the data, such as contiguity or proximity. However, connectivity does not
always need to hold for all regions, and in certain contexts it makes
sense to relax connectivity or to impose different types of spatial constraints. 

In this chapter we consider clustering techniques and regionalization methods which will
allow us to do exactly that. In the process, we will explore the characteristics
of neighborhoods in San Diego.
We will extract common patterns from the
cloud of multidimensional data that the Census Bureau produces about small areas
through the American Community Survey. We begin with an exploration of the
multivariate data about San Diego by suggesting some ways to examine the 
statistical and spatial distribution of the data before carrying out any
 clustering. Focusing on the individual variables, as well as their pairwise
associations, can help guide the subsequent application of clusterings or regionalizations. 

We then consider geodemographic approaches to clustering&mdash;the application
of multivariate clustering to spatially referenced demographic data.
Two popular clustering algorithms are employed: k-means and Ward's hierarchical method.
Mapping the spatial distribution of the resulting clusters 
reveals interesting insights on the socioeconomic structure of the San Diego
metropolitan area. We also see that in many cases, clusters are spatially 
fragmented. That is, a cluster may actually consist of different areas that are not
spatially connected. Indeed, some clusters will have their members strewn all over the map. 
This will illustrate why connectivity might be important when building insight
about spatial data, since these clusters will not at all provide intelligible regions. 
So, we then will move on to regionalization, exploring different approaches that
incorporate geographical constraints into the exploration of the social structure of San Diego.

## Data

The dataset we will use in this chapter comes from the American Community Survey
(ACS). In particular, we examine data at the Census Tract level in San Diego,
California in 2016. Let us begin by reading in the data as a GeoDataFrame and
exploring the attribute names.

```python
# Read file
db = geopandas.read_file('data/sd_tracts_acs_clean.shp')
# Print column names
db.columns
```

While the ACS comes with a large number of attributes we can use for clustering
and regionalization, we are not limited to the original variables at hand; we
can construct additional variables. This is particularly useful when
we want to compare areas that are not very similar in some structural
characteristic, such as area or population. For example, a quick look into the
variable names shows most variables are counts. For tracts of different sizes,
these variables will mainly reflect their overall population, rather than provide direct information
about the variables itself. To get around this, we will cast many of these count variables to rates,
and use them in addition to a subset of the original variables. 
Together, this set of constructed and received variables will to
will be used for our clustering and regionalization.

```python
# Pull out total house units
total_units = db['Total Ho_1']
# Calculate percentage of renter occupied units
pct_rental = db['Renter Occ'] / (total_units + (total_units==0)*1)

# Pull out total number of households
total_hh = db['Total Hous']
# Calculate percentage of female households
pct_female_hh = db['Female hou'] / (total_hh + (total_hh==0)*1)

# Calculate percentage of population with a bachelor degree
pct_bachelor = db["Bachelor's"] / (db['Total Popu'] + (db['Total Popu']==0)*1)
# Assign newly created variables to main table `db`
db['pct_rental'] = pct_rental
db['pct_female_hh'] = pct_female_hh
db['pct_bachelor'] = pct_bachelor
# Calculate percentage of population white
db['pct_white'] = db["White"] / (db['Total Popu'] + (db['Total Popu']==0) * 1)
```

To make things easier later on, let us collect the variables we will use to
characterize Census tracts. These variables capture different aspects of the socio-
economic reality of each area and, taken together, they provide a comprehensive
characterization of San Diego as a whole:

```python
cluster_variables =  ['Median Val',   # Median house value
                      'pct_white',    # Percent of tract population that is white
                      'pct_rental',   # Percent of households that are rented
                      'pct_female_hh',# Percent of female-led households 
                      'pct_bachelor', # Percent of tract population with a Bachelors degree
                      'Median Num',   # Median number of rooms in the tract's households
                      'Gini index',   # Gini index measuring tract wealth inequality
                      'Med Age',      # Median age of tract population
                      'Travel tim'    # Travel time to work 
                      ]
```

### Exploring the data

Now let's start building up our understanding of this
dataset through both visual and summary statistical measures.

We will start by
looking at the spatial distribution of each variable alone.
This will help us draw a picture of the multi-faceted view of the tracts we
want to capture with our clustering. Let's use choropleth maps for the
nine attributes and compare these choropleth maps side-by-side:

```python
f, axs = plt.subplots(nrows=3, ncols=3, figsize=(12, 12))
# Make the axes accessible with single indexing
axs = axs.flatten()
# Start a loop over all the variables of interest
for i, col in enumerate(cluster_variables):
    # select the axis where the map will go
    ax = axs[i]
    # Plot the map
    #db.plot(column=col, ax=ax, scheme='Quantiles', 
    #        linewidth=0, cmap='RdPu')
    choropleth(db, col, cmap='RdPu', ax=ax)
    # Remove axis clutter
    ax.set_axis_off()
    # Set the axis title to the name of variable being plotted
    ax.set_title(col)
# Display the figure
plt.show()
```

Many visual patterns jump out from the maps, revealing both commonalities as
well as differences across the spatial distributions of the individual variables.
Several variables tend to increase in value from the east to the west
(`pct_rental`, `Median Val`, `Median Num`, and `Travel tim`) while others
have a spatial trend in the opposite direction (`pct_white`, `pct_female_hh`,
`pct_bachelor`, `Med Age`). This is actually desirable; when variables have
different spatial distributions, each variable to contributes distinct 
information to the profiles of each cluster. However, if all variables display very similar 
spatial patterns, the amount of useful information across the maps is 
actually smaller than it appears, so cluster profiles may be much less useful as well.
It is also important to consider whether the variables display any
spatial autocorrelation, as this will affect the spatial structure of the
resulting clusters. 

Recall from chapter XXX that Moran's I is a commonly used
measure for global spatial autocorrelation. 
Let us get a quick sense to what
extent this is present in our dataset.
First, we need to build a spatial weights matrix that encodes the spatial
relationships in our San Diego data. We will start with queen contiguity:

```python
w_queen = Queen.from_dataframe(db)
```

As the warning tells us, observation `103` is an *island*, a disconnected observation
with no queen contiguity neighbors. To make sure that every observation
has at least one neighbor, we can combine the queen contiguity matrix with a
nearest neighbor matrix. This would ensure that every observation is neighbor 
of at least the observation it is closest to, plus all the areas with which 
it shares any border. Let's first create the `KNN-1 W`:

```python
w_k1 = KNN.from_dataframe(db, k=1)
```

Now we can combine the queen and nearest neighbor matrices into a single representation
with no disconnected observations. This full-connected connectivity matrix is the 
one we will use for analysis:

```python
w = Wsets.w_union(w_queen, w_k1)
```

As we ensured (thanks to the nearest neighbor connections),  `w` does not contain
any islands:

```python
w.islands
```

Now let's calculate Moran's I for the variables being used. This will measure
the extent to which each variable contains spatial structure:

```python
# Set seed for reproducibility
numpy.random.seed(123456)
# Calculate Moran's I for each variable
mi_results = [Moran(db[variable], w) for variable in cluster_variables]
# Display on table
table = pandas.DataFrame([(variable, res.I, res.p_sim) for variable,res 
                      in zip(cluster_variables, mi_results)],
                     columns=['Variable', "Moran's I", 'P-value'])\
          .set_index('Variable')
table
```

Each of the variables displays significant positive spatial autocorrelation,
suggesting that Tobler's law is alive and well in the socioeconomic geography of San
Diego County. This means we also should expect the clusters we find will have
a non random spatial distribution. In particular, we would expect clusters to have
a modest amount of spatial coherence in addition to the coherence in their profiles,
since there is strong positive autocorrelation in all of the input variables.

Spatial autocorrelation only describes relationships between a single observation at a time.
So, the fact that all of the clustering variables are positively autocorrelated does not tell us 
about the way the attributes covary over space. For that, we need to consider the
spatial correlation between variables. Here, we will measure this using the
bivariate correlation in the maps of covariates themselves.

Given the 9 maps, there are 36 pairs of maps that must be compared. This is too 
many maps to process visually, so we can turn to an alternative tool to
explicitly focus on the bivariate relations between each pair of attributes.

```python
_ = seaborn.pairplot(db[cluster_variables], kind='reg', diag_kind='kde')
```

Two different types of plots are contained in the scatterplot matrix. On the
diagonal are the density functions for the nine attributes. These allow for an
inspection of the overall morphology of the attribute's value distribution.
Examining these we see that our selection of variables includes those that are
negatively skewed (`pct_white` and `pct_female_hh`) as well as positively skewed
(`while median_val`, `pct_bachelor`, and `travel_tim`).

The second type of visualization lies in the off-diagonal cells of the matrix; 
these are bi-variate scatterplots. Each cell shows the association between one
pair of variables. Several of these cells indicate positive linear
associations (`med_age` Vs. `median_value`, `median_value` Vs. `Median Num`)
while other cells display negative correlation (`Median Val` Vs. `pct_rental`,
`Median Num` Vs. `pct_rental`, and `Med Age` Vs. `pct_rental`). The one variable
that tends to have consistenty weak association with the other variables is
`Travel tim`, and in part this appears to reflect its rather concentrated 
distribution as seen on the lower right diagonal corner cell.

## Geodemographic Clusters in San Diego Census Tracts

We now will move
beyond the implicitly bi-variate focus to consider the full multidimensional
nature of this data set. Geodemographic analysis is a form of multivariate
clustering where the observations represent geographical areas. The output
of these clusterings is nearly always mapped. Altogether, these methods use
multivariate clustering algorithms to construct a known number of
clusters ($k$), where the number of clusters is typically much smaller than the 
number of observations to be clustered. Each cluster is given a unique label,
and these labels are mapped. Using the clusters' profile and label, the map of 
labels can be interpreted to get a sense of the spatial distribution of 
sociodemographic traits. The power of (geodemographic) clustering comes
from taking statistical variation across several dimensions and compressing it
into a single categorical one that we can visualize through a map. To
demonstrate the variety of approaches in clustering, we will show two
distinct but very popular clustering algorithms: k-means and Ward's hierarchical method.

### K-means

K-means is probably the most widely used approach to
cluster a dataset. The algorithm groups observations into a
prespecified number of clusters so that that each observation is
closer to the mean of its own cluster than it is to the mean of any other cluster.
The k-means problem is solved by iterating between an assignment step and an update step. 
First, all observations are randomly assigned one of the $k$ labels. Next, the 
multivariate mean over all covariates is calculated for each of the clusters.
Then, each observation is reassigned to the cluster with the closest mean. 
If the observation is already assigned to the cluster whose mean it is closest to,
the observation remains in that cluster. This assignment-update process continues
until no further reassignments are necessary.

The nature of this algorithm requires us to select the number of clusters we 
want to create. The right number of clusters is unknown in practice. For
illustration, we will use $k=5$ in the `KMeans` implementation from
`scikit-learn`. To proceed, we first create a `KMeans` clusterer:

```python
# Initialise KMeans instance
kmeans = KMeans(n_clusters=5)
```

Next, we call the `fit` method to actually apply the k-means algorithm to our data:

```python
# Set the seed for reproducibility
numpy.random.seed(1234)
# Run K-Means algorithm
k5cls = kmeans.fit(db[cluster_variables])
```

Now that the clusters have been assigned, we can examine the label vector, which 
records the cluster to which each observation is assigned:

```python
k5cls.labels_
```

In this case, the second and third observations are assigned to cluster 4, while
the fourth and fifth observations have been placed in cluster 0. It is important
to note that the integer labels should be viewed as denoting membership only &mdash;
the numerical differences between the values for the labels are meaningless.
The profiles of the various clusters must be further explored by looking
at the values of each dimension. 

But, before we do that, let's make a map.

### Spatial Distribution of Clusters

Having obtained the cluster labels, we can display the spatial
distribution of the clusters by using the labels as the categories in a
choropleth map. This allows us to quickly grasp any sort of spatial pattern the 
clusters might have. Since clusters represent areas with similar
characteristics, mapping their labels allows to see to what extent similar areas tend
to have similar locations.
Thus, this gives us one map that incorporates the information of from all nine covariates.

```python
# Assign labels into a column
db['k5cls'] = k5cls.labels_
# Setup figure and ax
f, ax = plt.subplots(1, figsize=(9, 9))
# Plot unique values choropleth including a legend and with no boundary lines
db.plot(column='k5cls', categorical=True, legend=True, linewidth=0, ax=ax)
# Remove axis
ax.set_axis_off()
# Keep axes proportionate
plt.axis('equal')
# Add title
plt.title(r'Geodemographic Clusters (k-means, $k=5$)')
# Display the map
plt.show()
```

The map provides a useful view of the clustering results; it allows for
a visual inspection of the extent to which Tobler's first law of geography is
reflected in the multivariate clusters. Recall that the law implies that nearby
tracts should be more similar to one another than tracts that are geographically
more distant from each other. We can see evidence of this in
our cluster map, since clumps of tracts with the same color emerge. However, this
visual inspection is obscured by the complexity of the underlying spatial
units. Our eyes are drawn to the larger polygons in the eastern part of the
county, giving the impression that cluster 2 is the dominant cluster. While this
seems to be true in terms of land area (and we will verify this below), there is
more to the cluster pattern than this. Because the tract polygons are all 
different sizes and shapes, we cannot solely rely on our eyes to interpret 
the spatial distribution of clusters.

### Statistical Analysis of the Cluster Map

To complement the geovisualization of the clusters, we can explore the
statistical properties of the cluster map. This process allows us to delve
into what observations are part of each cluster and what their
characteristics are.
This gives us the profile of each cluster so we can interpret the meaning of the
labels we've obtained. We can start, for example, by
considering cardinality, or the count of observations in each cluster:

```python
# Group data table by cluster label and count observations
k5sizes = db.groupby('k5cls').size()
k5sizes
```

And we can get a visual representation of cardinality as well:

```python
_ = k5sizes.plot.bar()
```

There are substantial differences in the sizes of the five clusters, with two very
large clusters (0, 2), one medium sized cluster (4), and two small clusters (1,
3). Cluster 2 is the largest when measured by the number of assigned tracts.
This confirms our intuition from the map above, where we got the visual impression
that tracts in cluster 2 seemed to have the largest area. Let's see if this is 
the case. To do so we can use the `dissolve` operation in `geopandas`, which 
combines all tracts belonging to each cluster into a single
polygon object. After we have dissolved all the members of the clusters,
we report the total land area of the cluster:

```python
# Dissolve areas by Cluster, aggregate by summing, and keep column for area
areas = db.dissolve(by='k5cls', aggfunc='sum')['AREALAND']
areas
```

And, to show this visually:

```python
areas.plot.bar()
```

Our visual impression is confirmed: cluster 2 contains tracts that
together comprise 5,816,736,150 square meters (approximately 2,245 square miles),
which accounts for over half of the total land area in the county:

```python
areas[2]/areas.sum()
```

Let's move on to build the profiles for each cluster. Again, the profiles is what
provides the conceptual shorthand, moving from the arbitrary label to a meaningful
collection of observations with similar attributes. To build a basic profile, we can
compute the means of each of the attributes in every cluster:

```python
# Group table by cluster label, keep the variables used 
# for clustering, and obtain their mean
k5means = db.groupby('k5cls')[cluster_variables].mean()
k5means.T
```

We see that cluster 3, for example, is composed of tracts that have
the highest average `Median_val`, while cluster 2 has the highest level of inequality
(`Gini index`), and cluster 1 contains an older population (`Med Age`)
who tend to live in housing units with more rooms (`Median Num`).
Average values, however, can hide a great deal of detail and, in some cases,
give wrong impressions about the type of data distribution they represent. To
obtain more detailed profiles, we can use the `describe` command in `pandas`, 
after grouping our observations by their clusters:

```python
# Group table by cluster label, keep the variables used 
# for clustering, and obtain their descriptive summary
k5desc = db.groupby('k5cls')[cluster_variables].describe()
# Loop over each cluster and print a table with descriptives
for cluster in k5desc.T:
    print('\n\t---------\n\tCluster %i'%cluster)
    print(k5desc.T[cluster].unstack())
```

However, this approach quickly gets out of hand: more detailed profiles can simply
return to an unwieldy mess of numbers. A better approach to constructing
cluster profiles is be to draw the distributions of cluster members' data.
To do this we need to "tidy up" the dataset. A tidy dataset ([Wickham,
2014](https://www.jstatsoft.org/article/view/v059i10)) is one where every row is
an observation, and every column is a variable. Thus, a few steps are required 
to tidy up our labelled data:

```python
# Index db on cluster ID
tidy_db = db.set_index('k5cls')
# Keep only variables used for clustering
tidy_db = tidy_db[cluster_variables]
# Stack column names into a column, obtaining 
# a "long" version of the dataset
tidy_db = tidy_db.stack()
# Take indices into proper columns
tidy_db = tidy_db.reset_index()
# Rename column names
tidy_db = tidy_db.rename(columns={
                        'level_1': 'Attribute', 
                        0: 'Values'})
# Check out result
tidy_db.head()
```

Now we are ready to plot. Below, we'll show the distribution of each cluster's values
for each variable. This gives us the full distributional profile of each cluster:

```python
# Setup the facets
facets = seaborn.FacetGrid(data=tidy_db, col='Attribute', hue='k5cls', \
                  sharey=False, sharex=False, aspect=2, col_wrap=3)
# Build the plot from `sns.kdeplot`
_ = facets.map(seaborn.kdeplot, 'Values', shade=True).add_legend()
```

This allows us to see that, while some attributes such as the percentage of
female households (`pct_female_hh`) display largely the same distribution for
each cluster, others paint a much more divided picture (e.g. `Median Val`).
Taken altogether, these graphs allow us to start delving into the multidimensional 
complexity of each cluster and the types of areas behind them.

## Hierarchical Clustering

As mentioned above, k-means is only one clustering algorithm. There are
plenty more. In this section, we will take a similar look at the San Diego
dataset using another staple of the clustering toolkit: agglomerative
hierarchical clustering (AHC). Agglomerative clustering works by building a hierarchy of
clustering solutions that starts with all singletons (each observation is a single
cluster in itself) and ends with all observations assigned to the same cluster.
These extremes are not very useful in themselves. But, in between, the hierarchy
contains many distinct clustering solutions with varying levels of detail. 
The intuition behind the algorithm is also rather straightforward: 

1) begin with everyone as part of its own cluster; 
2) find the two closest observations based on a distance metric (e.g. euclidean); 
3) join them into a new cluster; 
4) repeat steps 2) and 3) until reaching the degree of aggregation desired. 

The algorithm is thus called "agglomerative"
because it starts with individual clusters and "agglomerates" them into fewer
and fewer clusters containing more and more observations each. Also, like with 
k-means, AHC does require the user to specify a number of clusters in advance.
This is because, following from the mechanism the method has to build clusters, 
AHC can provide a solution with as many clusters as observations ($k=n$),
or with a only one ($k=1$).

Enough of theory, let's get coding! In Python, AHC can be run
with `scikit-learn` in very much the same way we did for k-means in the previous
section. In this case, we use the `AgglomerativeClustering` class and again 
use the `fit` method to actually apply the clustering algorithm to our data:

```python
# Set seed for reproducibility
numpy.random.seed(0)
# Iniciate the algorithm
model = AgglomerativeClustering(linkage='ward', n_clusters=5)
# Run clustering
model.fit(db[cluster_variables])
# Assign labels to main data table
db['ward5'] =model.labels_
```

As above, we can check the number of observations that fall within each cluster:

```python
ward5sizes = db.groupby('ward5').size()
ward5sizes
```

Further, we can check the simple average profiles of our clusters:

```python
ward5means = db.groupby('ward5')[cluster_variables].mean()
ward5means.T
```

And again, we can create a plot of the profiles' distributions (after properly 
tidying up):

```python
# Index db on cluster ID
tidy_db = db.set_index('ward5')
# Keep only variables used for clustering
tidy_db = tidy_db[cluster_variables]
# Stack column names into a column, obtaining 
# a "long" version of the dataset
tidy_db = tidy_db.stack()
# Take indices into proper columns
tidy_db = tidy_db.reset_index()
# Rename column names
tidy_db = tidy_db.rename(columns={
                        'level_1': 'Attribute', 
                        0: 'Values'})
# Check out result
tidy_db.head()
```

```python
# Setup the facets
facets = seaborn.FacetGrid(data=tidy_db, col='Attribute', hue='ward5', \
                  sharey=False, sharex=False, aspect=2, col_wrap=3)
# Build the plot as a `sns.kdeplot`
_ = facets.map(seaborn.kdeplot, 'Values', shade=True).add_legend()
```

For the sake of brevity, we will not spend much time on the plots above.
However, the interpretation is analogous to that of the k-means example.

On the spatial side, we can explore the geographical dimension of the
clustering solution by making a map the clusters:

```python
db['ward5'] =model.labels_
# Setup figure and ax
f, ax = plt.subplots(1, figsize=(9, 9))
# Plot unique values choropleth including a legend and with no boundary lines
db.plot(column='ward5', categorical=True, legend=True, linewidth=0, ax=ax)
# Remove axis
ax.set_axis_off()
# Keep axes proportionate
plt.axis('equal')
# Add title
plt.title('Geodemographic Clusters (AHC, $k=5$)')
# Display the map
plt.show()
```

And, to make comparisons simpler, we can display both the k-means and the AHC
results side by side:

```python
db['ward5'] =model.labels_
# Setup figure and ax
f, axs = plt.subplots(1, 2, figsize=(12, 6))

ax = axs[0]
# Plot unique values choropleth including a legend and with no boundary lines
db.plot(column='ward5', categorical=True, cmap='Set2', 
        legend=True, linewidth=0, ax=ax)
# Remove axis
ax.set_axis_off()
# Keep axes proportionate
ax.axis('equal')
# Add title
ax.set_title('K-Means solution ($k=5$)')

ax = axs[1]
# Plot unique values choropleth including a legend and with no boundary lines
db.plot(column='k5cls', categorical=True, cmap='Set3',
        legend=True, linewidth=0, ax=ax)
# Remove axis
ax.set_axis_off()
# Keep axes proportionate
ax.axis('equal')
# Add title
ax.set_title('AHC solution ($k=5$)')

# Display the map
plt.show()
```

While we must remember our earlier caveat about how irregular polygons can 
baffle our visual intuition, a closer visual inspection of the cluster geography
suggests a clear pattern: although they are not identical, both clusterings capture
very similar overall spatial structure. Furthermore, both solutions slightly violate 
Tobler's law, since all of the clusters have disconnected components. The five
multivariate clusters in each case are actually composed of many disparate 
geographical areas, strewn around the map according only to the structure of the
data and not its geography. That is, in order to travel to
every tract belonging to a cluster, we would have to journey through
other clusters as well.

## Spatially Constrained Hierarchical Clustering

Fragmented clusters are not intrinsically invalid, particularly if we are
interested in exploring the overall structure and geography of multivariate
data. However, in some cases, the application we are interested in might
require that all the observations in a class be spatially connected. For
example, when detecting communities or neighborhoods (as is sometimes needed when
drawing electoral or census boundaries), they are nearly always distinct 
self-connected areas, unlike our clusters shown above. To ensure that clusters are
not spatially fragmented, we turn to regionalization.

Regionalization methods are clustering techniques that impose a spatial constraints
on clusters. In other words, the result of a regionalization algorithm contains clusters with
areas that are geographically coherent, in addition to having coherent data profiles. 
Effectively, this means that regionalization methods construct clusters that are 
all internally-connected; these are the *regions*. Thus, a regions' members must
be geographically *nested* within the region's boundaries.

This type of nesting relationship is easy to identify
in the real world. For example, counties nest within states, or, in the UK, 
local super output areas (LSOAs) nest within middle super output areas (MSOAs). 
The difference between these real-world nestings and the output of a regionalization
algorithm is that the real-world nestings are aggregated according to administrative principles, but regions' members are aggregated according to a statistical technique. In the same manner as the
clustering techniques explored above, these regionalization methods aggregate 
observations that are similar in their covariates; the profiles of regions are useful
in a similar manner as the profiles of clusters. But, in regionalization, the 
clustering is also spatially constrained, so the region profiles and members will
likely be different from the unconstrained solutions.

As in the non-spatial case, there are many different regionalization methods.
Each has a different way to measure (dis)similarity, how the similarity is used
to assign labels, how these labels are iteratively adjusted, and so on. However,
as with clustering algorithms, regionalization methods all share a few common traits.
In particular, they all take a set of input attributes and a representation of 
spatial connectivity in the form of a binary spatial weights matrix. Depending 
on the algorithm, they also require the desired number of output regions. For
illustration, we will take the AHC algorithm we have just used above, and apply 
an additional spatial constraint. In `scikit-learn`, this is done using
our spatial weights matrix as a `connectivity` option.
This will force the agglomerative algorithm to only allow observations to be grouped
in a cluster if they are also spatially connected:

```python
numpy.random.seed(123456)
model = AgglomerativeClustering(linkage='ward',
                                            connectivity=w.sparse,
                                            n_clusters=5)
model.fit(db[cluster_variables])
```

Let's inspect the output:

```python
db['ward5wq'] = model.labels_
# Setup figure and ax
f, ax = plt.subplots(1, figsize=(9, 9))
# Plot unique values choropleth including a legend and with no boundary lines
db.plot(column='ward5wq', categorical=True, legend=True, linewidth=0, ax=ax)
# Remove axis
ax.set_axis_off()
# Keep axes proportionate
plt.axis('equal')
# Add title
plt.title(r'Geodemographic Regions (Ward, $k=5$, Queen Contiguity)')
# Display the map
plt.show()

```

Introducing the spatial constraint results in fully-connected clusters with much
more concentrated spatial distributions. From an initial visual impression, it might
appear that our spatial constraint has been violated: there are tracts for both cluster 0 and
cluster 1 that appear to be disconnected from the rest of their clusters.
However, closer inspection reveals that each of these tracts is indeed connected
to another tract in its own cluster by very narrow shared boundaries.

### Changing the spatial constraint

The spatial constraint in regionalization algorithms is structured by the
spatial weights matrix we use. An interesting
question is thus how the choice of weights influences the final region structure.
Fortunately, we can directly explore the impact that a change in the spatial weights matrix has on
regionalization. To do so, we use the same attribute data
but replace the Queen contiguity matrix with a spatial k-nearest neighbor matrix,
where each observation is connected to its four nearest observations, instead
of those it touches.

```python
w = KNN.from_shapefile('data/sd_tracts_acs_clean.shp', k=4)
```

With this matrix connecting each tract to the four closest tracts, we can run 
another AHC regionalization:

```python
numpy.random.seed(1234567)
model = AgglomerativeClustering(linkage='ward',
                                            connectivity=w.sparse,
                                            n_clusters=5)
model.fit(db[cluster_variables])
```

And plot the final regions:

```python
db['ward5wknn'] = model.labels_
# Setup figure and ax
f, ax = plt.subplots(1, figsize=(9, 9))
# Plot unique values choropleth including a legend and with no boundary lines
db.plot(column='ward5wknn', categorical=True, legend=True, linewidth=0, ax=ax)
# Remove axis
ax.set_axis_off()
# Keep axes proportionate
plt.axis('equal')
# Add title
plt.title('Geodemographic Regions (Ward, $k=5$, four nearest neighbors)')
# Display the map
plt.show()
```

Even though we have specified a spatial constraint, the constraint applies to the
connectivity graph modeled by our weights matrix. Therefore, using k-nearest neighbors
to constrain the agglomerative clustering may not result in regions that are connected
according to a different connectivity rule, such as the queen contiguity rule used
in the previous section. However, the regionalization here is fortuitous; even though
we used the 4-nearest tracts to constrain connectivity, all but one of the clusters, 
cluster 4, is *also* connected according to our earlier queen contiguity rule. 

At first glance, this may seem counter-intuitive. We did specify the spatial
constraint, so our initial reaction is that the connectivity constraint is
violated. However, this is not the case, since the constraint applies to the
k-nearest neighbor graph, not the queen contiguity graph. Therefore, since tracts
in this solution are considered as connected to their four closest neighbors,
clusters can "leapfrog" over one another. Thus, it is important to recognize that
the apparent spatial structure of regionalizations will depend on how the 
connectivity of observations is modeled. 

## Max-P
We close this chapter with an alternative approach to spatially constrained clustering.
Here we focus on the `max-p` algorithm [(Duque, Anselin, Rey 2012)](https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1467-9787.2011.00743.x)
that treats the number of regions as endogenous, in contrast to the approaches thus far where the user ultimately has to specify the number of regions to form. In addition to the spatial adjacency matrix, the algorithm requires a `threshold` variable that specifies the minimum value that each region must satisfy.




<div class="alert alert-info" style="font-size:120%">
<b>NOTE</b>: <br>

Max-P is a stochastic algorithm so the results may be different on different platforms. The following section includes values/results that may differe slightly upon reruns of the notebook.

This section has not yet been included in the Book, but is intended to illustrate Max-p.

The Region module is now at end-of-life and new work on max-p has moved into the pysal/spopt module which is under development.

</div>


We first import the `MaxPRegionsHeu` class from the `region` module:

```python
from region.max_p_regions.heuristics import MaxPRegionsHeu
```

We will continue with the same variables for clustering that we have been using:

```python
db[cluster_variables].head()
```

Our threshold variable used to specify the floor constraint is going to be the total population of each tract:

```python
db['Total Popu']
```

First we instantiate an instance of the model:

```python
model = MaxPRegionsHeu(random_state=123)
```

As this algorithm currently only permits the adjacency to be based on rook or queen, we need to drop one tract since we previous saw that this tract is an isolate:

```python
db.iloc[103]
```

```python
db1 = db.drop(db.index[103])
```

With the new dataframe in hand we can fit the model:

```python
model.fit_from_geodataframe(db1, cluster_variables, 'Total Popu', 15000, contiguity='queen')
```

We have specified that the floor constraint is a region will consist of at least
15000 total population. This is the reason we dropped tract 103, since its
population is insufficient to form a single region and thus if it were included
the model would fail to find a feasible solution.

We also specify `queen` be used to form the adjacency matrix from the geodataframe.

```python
model.labels_
```

Once the model is fit, a new attribute `labels_` is created which denotes the region id that the tract was assigned to. 

We can find the number of neighborhoods/regions as the maximum value of this attribute:

```python
model.labels_.max()
```

We can also visualize the resulting neighborhoods:

```python
db1['maxp_15000'] = model.labels_
# Setup figure and ax
f, ax = plt.subplots(1, figsize=(9, 9))
# Plot unique values choropleth including a legend and with no boundary lines
db1.plot(column='maxp_15000', categorical=True, legend=True, linewidth=0, ax=ax)
# Remove axis
ax.set_axis_off()
# Keep axes proportionate
plt.axis('equal')
# Add title
plt.title('Geodemographic Regions (Max-P, $15000$, queen)')
# Display the map
plt.show()
```

Note that visualization of the neighborhoods is challenging since the number of neighborhoods is large and differentiation via color relies on the categorical color scheme which gets exhausted and the same color can be applied to different clusters.

So we can explore our solution further to ensure the model behaves as we expect. First,
we examine the threshold constraint which requires each neighborhood (group of tracts) to have at least 15000 people:

```python
thresh = db1[['Total Popu', 'maxp_15000']].groupby(by='maxp_15000').sum()
```

The smallest neighborhood in this regard satisfies this constraint:

```python
thresh.min()
```

While the size of the largest region is:

```python
thresh.max()
```
```python
db1['tracts'] = numpy.ones(db1.shape[0],)
counts = db1[['Total Popu', 'maxp_15000','tracts']].groupby(by='maxp_15000').count()
counts
```

```python
counts.tracts.max()
```

```python
counts.tracts.min()
```

```python
from pysal.lib.weights import block_weights
```

```python
adj_check = block_weights(db1['maxp_15000'])
```

```python
adj_check.n
```

```python
(counts.tracts==1).sum()

```

We first check if there are islands, which are isolates, or neighborhoods composed of a single tract:

```python
adj_check.islands
```

```python
db1.iloc[adj_check.islands]
```

These have tract populations that allows them to serve as individual regions in a feasible solution.

```python
isolates = db1[['maxp_15000']].iloc[adj_check.islands]
```

And we see that the regions these tracts belong to are indeed isolates.

```python
isolates
```

```python
db1[db1.maxp_15000.isin(isolates.maxp_15000)]
```

Next, if the model has respected the contiguity constraint, the adjacency graph should have as many connected components as we do neighborhoods. We can check this with the aid of the `networkx` package:

```python
import networkx
```

We first obtain the graph representation of our spatial weights object:

```python
g = adj_check.to_networkx()
```
and then pass that to find the number of connected components:
```python
nc = networkx.number_connected_components(g)
print(nc)
print (nc == (1 + model.labels_.max()) )
```
Note we add 1 since python is 0-offset.

## Conclusion

Overall, clustering and regionalization are two complementary tools to reduce the
complexity in multivariate data and build better understandings of the spatial structure 
of data. Often, there is simply too much data to examine every variables' map and its
relation to all other variable maps. 
Thus, clustering reduces this complexity into a single conceptual shorthand by which 
people can easily describe complex and multifaceted data. 
Clustering constructs groups of observations (called *clusters*)
with coherent *profiles*, or distinct and internally-consistent 
distributional/descriptive characteristics. 
These profiles are the conceptual shorthand, since members of each cluster should
be more similar to the cluster at large than they are to any other cluster. 
Many different clustering methods exist; they differ on how the "cluster at large" 
is defined, and how "similar" members must be to clusters, or how these clusters
are obtained.
Regionalization is a special kind of clustering with an additional geographic requirement. 
Observations should be grouped so that each spatial cluster,
or *region*, is spatially-coherent as well as data-coherent. 
Thus, regionalization is often concerned with connectivity in a contiguity 
graph for data collected in areas; this ensures that the regions that are identified
are fully internally-connected. 
However, since many regionalization methods are defined for an arbitrary connectivity structure,
these graphs can be constructed according to different rules as well, such as the k-nearest neighbor graph. 

In this chapter, we discussed the conceptual basis for clustering and regionalization, 
as well showing why clustering is done. 
Further, we have shown how to build clusters using spatial data science packages, 
and how to interrogate the meaning of these clusters as well.
More generally, clusters are often used in predictive and explanatory settings, 
in addition to being used for exploratory analysis in their own right.
Clustering and regionalization are intimately related to the analysis of spatial autocorrelation as well,
since the spatial structure and covariation in multivariate spatial data is what
determines the spatial structure and data profile of discovered clusters or regions.
Thus, clustering and regionalization are essential tools for the spatial data scientist.

## Questions

1. What disciplines employ regionalization? Cite concrete examples for each discipline you list.
2. Contrast and compare  the concepts of *clusters* and *regions*?
3. In evaluating the quality of the solution to a regionalization problem, how might traditional measures of cluster evaluation be used? In what ways might those measures be limited and need expansion to consider the geographical dimensions of the problem?
4. Discuss the implications for the processes of regionalization that follow from the number of *connected components* in the spatial weights matrix that would be used.
5. True or false: The average silhouette score for a spatially constrained solution will be no larger than the average silhouette score for an unconstrained solution. Why, or why not? (add reference and  or explain silhouette)
6. Consider two possible weights matrices for use in a spatially constrained clustering problem. Both form a single connected component for all the areal units. However, they differ in the sparsity of their adjacency graphs (think Rook being more dense than Queen graphs). 
    a. How might the sparsity of the weights matrix affect the quality of the clustering solution?
    b. Using `pysal.lib.weights.higher_order`, construct a second-order adjacency matrix of the weights matrix used in this chapter. 
    c. Compare the `pct_nonzero` for both matrices. 
    d. Rerun the analysis from this chapter using this new second-order weights matrix. What changes? 
7. The idea of spatial dependence, that near things tend to be more related than distant things, is an extensively-studied property of spatial data. How might solutions to clustering and regionalization problems change if dependence is very strong and positive? very weak? very strong and negative? 
8. In other areas of spatial analysis, multilevel models [1] recognize that sometimes, geographical regions are more similar internally than they are externally. That is, two observations in the same region are probably more similar than two observations in different regions. If this kind of dependence is very strong, what would happen to clustering and regionalization solutions?
9. Using a spatial weights object obtained as `w = pysal.lib.weights.lat2W(20,20)`, what are the number of unique ways to partition the graph into 20 clusters of 20 units each, subject to each cluster being a connected component? What are the unique number of possibilities for `w = pysal.lib.weights.lat2W(20,20, rook=False)` ?

## References

[1] Gelman, A., & Hill, J. (2006). Data analysis using regression and multilevel/hierarchical models. Cambridge university press.

---

<a rel="license" href="http://creativecommons.org/licenses/by-nc-nd/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-nd/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-nd/4.0/">Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License</a>.

