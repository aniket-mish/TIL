# Linear Regression with Gradient Descent


In `scikit-learn` use `SGDRegressor` for LR with SGD. If you use `LinearRegression`, it uses OLS.

```python

class LinearRegression:
    def __init__(self, lr=0.01, epochs=10):
        self.lr = lr # learning rate
        self.epochs = epochs # number of epochs

    def fit(self, X, y):
        # number of observations and features
        self.observations, self.features = X.shape

        # initialize weights and bias terms
        self.w = np.zeros(self.features)
        self.b = 0

        # read data
        self.X = X
        self.y = y

        # use gradient descent to update weights
        for i in range(self.epochs):
            self.update_weights()

        return self # w and b are updated in-place
    

    def update_weights(self):
        # predict
        y_pred = self.predict(self.X)

        # compute gradients
        grad_w = -2 * np.dot(self.X.T, (self.y - y_pred)) / self.observations
        grad_b = -2 * np.dot(self.y - y_pred) / self.observations

        # update w and b
        self.w = self.w - self.lr * grad_w
        self.b = self.b - self.lr * grad_b

        return self
        

    def predict(self, X):
        # use current parameters to predict
        return X.dot(self.w) + self.b

```