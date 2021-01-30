# Problem

Construct a classifier that predicts whether the tumor is malignant (M), or benign (B) using methods of unsupervised learning and data from accompanying file data.csv. It is known there are more benign observations.

Features are computed from a digitized image of a fine needle aspirate (FNA) of a breast mass. They describe characteristics of the cell nuclei present in the image.

# Data

data.csv is a database containing data on 569 breast masses. The database contains the following information:

```{r echo=FALSE, warning = FALSE}
library(kableExtra)
library(glue)

table <- rbind(c(1, "id", "observation identification label"), c(2, "radius", "mean of distances from center to points on the perimeter"), c(3, "texture", "standard deviation of gray-scale values"), c(4, "perimeter", "perimeter of the mass"), c(5, "area", "area of the mass"), c(6, "smoothness", "local variation in radius lengths"), c(7, "compactness", "$\\dfrac{perimeter^2}{area} - 1$"), c(8, "concavity", "severity of concave portions of the contour"), c(9, "concave points", "number of concave portions of the contour"), c(10, "symmetry", "symmetry of the mass"), c(12, "fractal dimension", " \"coastline approximation\" - 1"))
knitr::kable(table, format = "latex", escape = TRUE)
table %>%
  kbl() %>%
  kable_styling()
```

# One possible solution

```{r}
data <- read.csv("data.csv") #loading the data
id <- data$id # saving the id column, since we need it for the result data frame
data <- data[, -c(1)] #removing the id column from our data
summary(data)

set.seed(123)
# our data need to be scaled, especially for the PCA algorithm
data.scaled <- as.data.frame(scale(data))
```


## Principal Component Analysis (PCA)

PCA is the algorithm for dimension-reduction. It returns linear transformation of our data in such way that the most variance is preserved in less predictors.
It needs to be performed on scaled data!

```{r}
library(factoextra) # used for pretty plots

pc <- princomp(data.scaled) # pca model
summary(pc)
fviz_eig(pc) # plots percentage of explained variance via the components

data.PCA <- data.frame(pc$scores[, c(1, 2, 3, 4)])
```

In the summary and plot of the PCA model we can see that > 80% of the variance is explained by the first 4 principal components. That is the reason why we will continue to work on data.PCA.  

## K-means

We can compute k-means in R with the kmeans function. Here, we will group the data into two clusters (centers = 2).

Firstly, we are going to show that the number of clusters that minimizes the distance within the clusters is exactly 2, since k = 2 is the number of clusters we need (mass is B or M), using the silhouette method.
Silhouette refers to a method of interpretation and validation of consistency within clusters of data. The technique provides information on how well each object has been classified.
We can use the silhouette function in the cluster package to compute the average silhouette width. The following code computes this approach for 2-10 clusters. The results show that 2 clusters maximize the average silhouette values.

Furthermore, the kmeans function also has a nstart option that attempts multiple initial configurations and reports on the best one. Eg. adding nstart = 20 will generate 20 initial configurations. This approach is often recommended.

```{r}
# library(cluster)
# 
# silhouette <- rep(0, 9)
# for (i in 2:10) {
#   model = kmeans(data.scaled, centers = i, nstart = 20)
#   silhouettes = silhouette(model$cluster, dist = dist(data.scaled))
#   silhouette[i] = mean(silhouettes[, 3])
# }
# plot(silhouette, type = 'b')
# abline(v = which.max(silhouette), lty = 2)

# the commented code above can be executed using the fviz_nbclust
# function from factoextra library
fviz_nbclust(data, kmeans, method = "silhouette")
```

After computing our k-means model, we are going to reformat our clusters' names, since we want the output in the form M/B, rather than 1/2. Here, we are using the information that there is more benign observations.

```{r}
model.kmeans.PCA = kmeans(data.PCA, centers = 2, nstart = 20) # computing kmeans model

# model.kmeans$size is a vector of length 2, indicating the sizes of the clusters.
# The first number is the number of observations labeled as "1",
# and the second number is the number of observations labeled as "2"
if(model.kmeans.PCA$size[1] > model.kmeans.PCA$size[2]){
  model.kmeans.PCA$cluster[model.kmeans.PCA$cluster == 1] <- "B"
  model.kmeans.PCA$cluster[model.kmeans.PCA$cluster == 2] <- "M"
}
if(model.kmeans.PCA$size[1] < model.kmeans.PCA$size[2]){
 model.kmeans.PCA$cluster[model.kmeans.PCA$cluster == 1] <- "M"
 model.kmeans.PCA$cluster[model.kmeans.PCA$cluster == 2] <- "B"
}
```

In the end, we are going to write the results in the .csv file.

```{r}
result <- as.data.frame(cbind(id, model.kmeans.PCA$cluster))
names(result) <- c("id", "prediction")
write.csv(result, "result.csv")
```



## Visualize clusters

We can visualize our results by using fviz_cluster from factoextra library. This provides a nice illustration of the clusters. If there are more than two variables fviz_cluster will perform principal component analysis (PCA) and plot the data points according to the first two principal components, since they explain the majority of the variance.

```{r}
fviz_pca_ind(pc,
             geom.ind = "point",
             pointshape = 21,
             pointsize = 2,
             fill.ind = model.kmeans.PCA$cluster,
             palette = c("#2E9FDF", "#00AFBB"),
             addEllipses = TRUE,
             legend.title = "Prediction") + 
  ggtitle("2D PCA-plot from 30 feature dataset") +
  theme(plot.title = element_text(hjust = 0.5))

```

