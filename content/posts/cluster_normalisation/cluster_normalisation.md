---
title: "Feature scaling for clustering"
date: 2017-11-28T01:00:00Z
featured_image: '/posts/cluster_normalisation/the-murmuration.jpg'
author: "James"
summary: "How and why to normalise data before clustering"
tags: ["R"]
draft: true
---

Introduction
============

This is a demonstration of the importance of normalising (also known as
standardising) data with distance based clustering techniques like
K-means. I was motivated by an answer in the Statistics StackExchange
community and have translated this from Python into R:
[Cross-Validated](https://stats.stackexchange.com/questions/89809/is-it-important-to-scale-data-before-clustering)
translated into R.

There are several types of 'cluster models', each of which contain
several algorithms. These include Centroid, Hierarchical, distribution
and density based models. This post will focus on the most common
algorithm: K-means, the architypal centroid-model based algorithm.

For K-means, as a centroid based cluster model, each cluster is
represented by a single central mean vector (x,y) that minimises the sum
of squared residuals. As this seeks to minimise the Euclidian distance,
there are a number of assumptions that result from this:

1.  Spherical Clusters
    -   Clusters must be separable such that the mean value (centroid)
        converges to the centre
    -   Clusters cannot be nested

2.  Similar scales
    -   Clusters have similar variance

3.  Similar sized clusters
    -   Unbalanced sizes of clusters gives greater weighting to the
        larger clusters

The Euclidean distance is used here as I have continuous numerical
variables. In this example the two sets of data will on different scales
and will thus differ in their variances.

Packages
--------

    library(dplyr) # Data Manipulation
    library(ggplot2) # Data Visualisation
    library(ggExtra) # Adding marginal plots to ggplots

Simulating the Data
===================

    # Data along variable X to have single distribution with large range
    x <- rnorm(1000) * 10 + 50

    # Data along Y to have two distributions with a smaller range
    y <- c(rnorm(500) + 10, rnorm(500) + 15)

    # Binding the data together into a dataframe
    df <- as.data.frame(cbind(x,y))
    head(df)

    ##          x         y
    ## 1 62.04969  9.818573
    ## 2 46.94655 10.573686
    ## 3 63.55241 10.463015
    ## 4 45.10800  9.276210
    ## 5 27.84014  8.870390
    ## 6 33.45776 11.068927

In this bi-variate example, the data along the x-axis is drawn from a
normal distribution and has a single modal value.

A bi-modal distribution was created on the y-axis; the data for each of
these distributions were drawn from different normal distributions,
creating a separation between clusters.

The range of the X-variable is larger than the range of Y.

    ggExtra::ggMarginal(
        ggplot(df, aes(x = x, y = y)) +
            geom_point() + theme_bw(),
    type = "histogram", fill = "lightgrey"
    )

![original data](/posts/cluster_normalisation/images/original_data.png)

Data Analysis and Results
=========================

K-means on un-scaled data
-------------------------

    # Perform K-means Clustering
    km <- kmeans(df, #features to be included in cluster model
                 2, # number of clusters
                 nstart = 20 # number of repeats
                 )

    #Create Dataframe of centroid locations
    centroids <- data.frame(Centroid = factor(seq(1:2)),
                            x = km$centers[,1],
                            y = km$centers[,2])

    # Plot clusters on data with centroids
    ggplot(df, aes(x = x, y = y)) +
        geom_point(aes(color = as.factor(km$cluster)), alpha = 0.5, pch = 19) +
        geom_point(data = centroids, aes(x = x, y = y, colour = Centroid), cex = 5, pch = 19) +
        guides(color = FALSE) + # remove legend
        theme_bw()

![unscaled clustering](/posts/cluster_normalisation/images/unscaled_clustering.png)

The algorithm has not clustered the data as we might have expected.
R-plots default to maximising the screen space and will distort each
axis for a better visualisation, and with the plot in this format it can
be difficult to see why the clusters have formed around these centroids.

    ggplot(df, aes(x = x, y = y)) +
        geom_point(aes(color = as.factor(km$cluster)), pch = 19, alpha = 0.5) +
        geom_point(data = centroids, aes(x = x, y = y, colour = Centroid), cex = 5, pch = 19) +
        guides(color = FALSE) + # remove legend
        coord_fixed() + # maintain correct aspect ratio (assuming change in x is same as change in y)
        theme_bw()

![unscaled clustering](/posts/cluster_normalisation/images/unscaled_aspect.png)

This plot now maintains the aspect ratio, creating representative
distances between points.

K means iteratively optimises the clusters by minimising the Sum of
Squared Errors (SSE), i.e. the distances between the centroids and the
data points, similar to linear regression.

This assumes that the variance of each of the variables are the same.
With the large difference in the scales of each feature, this is not the
case:

    df %>%
        summarise(range_x = max(x) - min(x),
                  range_y = max(y) - min(y),
                  var_x = var(x),
                  var_y = var(y)) %>%
        round(., 2) # round to 2 d.p.

    ##   range_x range_y var_x var_y
    ## 1   60.03   10.75 94.46  6.96

The data in this case are on different scales and thus more weighting is
given to the feature with the largest range.

Feature normalisation
---------------------

Normilisation eliminates the problem of different scales by bringing all
values between 0 and 1. There are various ways to scale data, here I
have used Unity based normalisation.

![Unity-based normilisation equation](https://latex.codecogs.com/gif.latex?x%5Cprime%20%3D%20%5Cfrac%7Bx%20-%20min%28x%29%7D%7Bmax%28x%29%20-%20min%28x%29%7D)


Where: *x* = data vector and *x*′ = normalised x

    # Create Feature Scaling function that takes a numeric vector.
    scaling <- function(x){
        (x - min(x)) / (max(x) - min(x))
    }

    # new dataframe with scaled values
    df %>%
        dplyr::mutate(x_scaled = scaling(df$x),
                      y_scaled = scaling(df$y)) %>%
        dplyr::select(x_scaled, y_scaled) -> df_scaled

    head(df_scaled)

    ##    x_scaled  y_scaled
    ## 1 0.7101954 0.2747678
    ## 2 0.4585957 0.3449835
    ## 3 0.7352288 0.3346926
    ## 4 0.4279676 0.2243350
    ## 5 0.1403063 0.1865991
    ## 6 0.2338890 0.3910346

    ggplot(data = df_scaled,
           mapping = aes(x = x_scaled, y = y_scaled)
           ) +
        geom_point() +
        theme_bw()

![normalised data](/posts/cluster_normalisation/images/normalised_data.png)

K-means on Scaled Data
----------------------

Now both features are on comparable scales, on which the Euclidean
distance can be measured to create a new K-means clustering model.

Results are plotted on the original scales for interpretability.

One issue with an iterative process such as k-means, is that the choice
of the initial starting point can influence the assignment of points to
clusters. The process is guaranteed to only find local optimums. This
can be accounted for by repeating the clustering mutiple times with
different randomly set initilisation values and choosing the result with
the lowest error rate. This is done using the `nstart = n` argument in
`kmeans()` function.

    # run k-means on scaled data
    km_scaled <- kmeans(df_scaled, 2, nstart = 20)

    # create an unscale function to return values back to original scale
    unscale <- function(x,y){
        x * ((max(y) - min(y))) + min(y)
    }

    centroids <- data.frame(Centroid = factor(seq(1:2)),
                            x = unscale(km_scaled$centers[,1], df$x),
                            y = unscale(km_scaled$centers[,2], df$y))

    ggplot(df, aes(x = x, y = y)) +
        geom_point(alpha = 0.5, pch = 19, aes(color = as.factor(km_scaled$cluster))) +
        geom_point(data = centroids, aes(x = x, y = y, colour = Centroid), cex = 5, pch = 19) +
        guides(color = FALSE) + # remove legend
        theme_bw()

![normalised clustering](/posts/cluster_normalisation/images/normalised_clustering.png)

The impact of feature scaling can be seen. this now satisfies the
assumptions of the Ordinary Least Squares, that minimises the sum of
squared errors, resulting in a more intuitive clustering.

Session Information
===================

    sessionInfo()

    ## R version 3.4.2 (2017-09-28)
    ## Platform: x86_64-w64-mingw32/x64 (64-bit)
    ## Running under: Windows 10 x64 (build 16299)
    ##
    ## Matrix products: default
    ##
    ## locale:
    ## [1] LC_COLLATE=English_United Kingdom.1252
    ## [2] LC_CTYPE=English_United Kingdom.1252   
    ## [3] LC_MONETARY=English_United Kingdom.1252
    ## [4] LC_NUMERIC=C                           
    ## [5] LC_TIME=English_United Kingdom.1252    
    ##
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ##
    ## other attached packages:
    ## [1] bindrcpp_0.2       ggExtra_0.7        ggplot2_2.2.1.9000
    ## [4] dplyr_0.7.4       
    ##
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_0.12.14     knitr_1.17       bindr_0.1        magrittr_1.5    
    ##  [5] munsell_0.4.3    xtable_1.8-2     colorspace_1.3-2 R6_2.2.2        
    ##  [9] rlang_0.1.4      plyr_1.8.4       stringr_1.2.0    tools_3.4.2     
    ## [13] grid_3.4.2       gtable_0.2.0     miniUI_0.1.1     htmltools_0.3.6
    ## [17] lazyeval_0.2.1   yaml_2.1.14      assertthat_0.2.0 rprojroot_1.2   
    ## [21] digest_0.6.12    tibble_1.3.4     shiny_1.0.5      mime_0.5        
    ## [25] glue_1.2.0       evaluate_0.10.1  rmarkdown_1.8    labeling_0.3    
    ## [29] stringi_1.1.6    compiler_3.4.2   scales_0.5.0     backports_1.1.1
    ## [33] httpuv_1.3.5     pkgconfig_2.0.1