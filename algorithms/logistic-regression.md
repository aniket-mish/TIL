# Logistic Regression

Its similar to linear regression with one extra step to turn the real-valued predictions into binary values


```python
class LogisticRegression:
    def __init__(self, lr=0.01, epochs=10):
        self.lr = lr
        self.epochs = epochs

    def fit(self, X, y):
        # number of observations and features
        self.observations, self.features = X.shape
        
        # initialize weights and bias terms
        self.w = np.zeros(self.features)
        self.b = 0

        # read data
        self.X = X
        self.y = y

        # use gradient descent to update parameters
        for i in range(self.epochs):
            self.update_weights()

        return self

    def update_weights(self):
        # use parameters to predict
        y_pred = self.predict(self.X)

        # calculate gradients
        grad_w = -np.dot(self.X.T * (y_pred - self.y)) / self.observations
        grad_b = -np.dot(self.y - y_pred) / self.observations

        # update parameters
        self.w = self.w - self.lr * grad_w
        self.b = self.b - self.lr * grad_b

        return self

    def sigmoid(self, z):
        return 1 / (1 + np.exp(-z))

    def predict(self, X):
        # linear
        z = X.dot(self.w) + self.b
        # transform to [0, 1] using sigmoid 
        p = self.sigmoid(z)
        # decide based on a threshold
        return np.where(p < 0.5, 0, 1)

```