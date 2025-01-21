# K-Nearest Neighbor

This is a non-parametric model with a time complexity of O(n * d) where n is the number of training examples and d is the number of features.

In modern systems, ANN lookups are really important. Look up HNSW algorithm.


```python
class KNNClassifier:
    def __init__(self, k=3):
        self.k = k

    def fit(self, X_train, y_train): # we only read the data and keep it in memory
        # read train data
        self.X_train = X_train
        self.y_train = y_train

        self.n_train = X_train.shape[0]

        return self

    def predict(self, X_test): # computationally expensive
        # read test data
        self.X_test = X_test

        self.n_test = X_test.shape[0]

        y_pred = np.zeros(self.n_test)

        # iterate through all points in test data
        for point in range(self.n_test):
            # get knn points for a given point
            p = X_test[point]
            neighbors = self.get_neighbors(p) # looping over all the training data to calculate pairwise distances and get sorted k labels

            # take majority
            y_pred[point] = mode(neighbors)[0][0]
        
        return y_pred

    def distance(self, d1, d2):
        # euclidean distance between d1 and d2
        return np.sqrt(np.sum((d1 - d2) ** 2))

    def get_neighbors(self, d):
        # compute distance
        distances = np.zeros(self.n_train)

        # iterate through all points in training data
        for point in range(self.n_train):
            distance = self.distance(d, self.X_train[point])
            distances[point] = distance

        # sort using distances
        _, y_train_sorted = zip(*sorted(zip(distances, y_train)))

        # return top-k
        return y_train_sorted[:, self.k]
```