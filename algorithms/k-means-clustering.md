# K-Means Clustering


```python
class KMeans:
    def __init__(self, n_clusters=4, epochs=10):
        self.n_clusters = n_clusters
        self.epochs = epochs

    def fit(self, X_train): # takes only feature array and not labels as we need to find structure of data
        # read data
        self.X_train = X_train

        # number of observations
        self.n_train = X_train.shape[0]

        # initialize random centroids within the bounds of the training data
        min_, max_ = np.min(self.X_train, axis=0), np.max(self.X_train, axis=0)
        centroids = [np.random.uniform(min_, max_) for _ in range(self.n_clusters)]

        # train
        for i in range(self.epochs):
            centroids = self.update_centroids(centroids)

        # save final centroids
        self.centroids = centroids
        return self

    def update_centroids(self, centroids):
        # store cluster labels
        clusters = np.zeros(self.n_train)

        # assign each data point to the closest cluster
        for i in range(self.n_train):
            point = self.X_train[i]
            distance = [self.get_distance(point, centroid) for centroid in centroids]

            # index of closest centroid is cluster label
            clusters[i] = np.argmin(distance)

        # update centroids by taking average of points in the given cluster
        for i in range(self.n_clusters):
            # get all points assigned to given cluster
            points = self.X_train[np.array(clusters) == i]

            # update new centroids
            centroids[i] = points.mean(axis=0)

        return centroids

    def get_distance(self, d1, d2):
        # euclidean distance
        return np.sqrt(np.sum((d1 - d2) ** 2))

    def predict(self, X_test):
        # read data
        self.X_test = X_test

        self.n_test = X_test.shape[0]

        clusters = np.zeros(self.n_test)

        # assign data points to closest centroid
        for i in range(self.n_test):
            point = self.X_test[i]
            distances = [self.get_distance(point, centroid) for centroid in self.centroids]
            clusters[i] = np.argmin(distances)

        # return predicted clusters
        return clusters
```