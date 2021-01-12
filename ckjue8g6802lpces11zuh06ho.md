## Heroku: Deploy a dockerized Flask ML app

> In this post we will deploy a simple dockerized Machine learning application written with the Flask micro-framework to the Heroku (PaaS), using one of the leading CI/CD tools, which is Gitlab!, all used tools are free and no fees in achieving this tutorial. 

# Tools to know 

This the list of tools that we need to achieve the aim of this tutorial alongside some bash scripting knowledge 

|Tools|Definition|
|-|-|
| **Flask** | is a micro web framework written in Python. It is classified as a micro-framework because it does not require specific tools or libraries. It has no database abstraction layer, form validation, or any other existing third-party library components that provide common functions. [Go farther with flask](https://www.fullstackpython.com/flask.html)|
| **Heroku** | is a container-based cloud Platform as a Service (PaaS). Developers use Heroku to deploy, manage, and scale modern apps. The platform is elegant, flexible, and easy to use, offering developers the simplest path to getting their apps to market. [Go farther with heroku](https://devcenter.heroku.com/start)| 
|**GitLab**  | is a web-based DevOps lifecycle tool that provides a Git-repository manager providing wiki, issue-tracking, and continuous integration and deployment pipeline features, using an open-source license. [Explore Gitlab documentation](https://docs.gitlab.com/)|

# Code base 

Before starting let's understand what we will dockerize here! This application is a simple machine learning that exposes two routes : 

* **/kmeans**: This route accepts a JSON data format with two entries. The first one is the number of clusters you want the classifier to consider, the second one is the text that we will group into the specified number of cluster, each group will contain words that are similar on the meaning or has the same area like **people**, **building**, **civilization** and so on.

* **/ads **: This route accepts also  JSON data format that contains also two entries which are the **age** and the **annual salary**, the result will be if this person can buy a product on which he saw it on a social media. 

The app used Two ML algorithms associated with routes :

1. **KMEANS** : [Learn more](https://towardsdatascience.com/understanding-k-means-clustering-in-machine-learning-6a6e67336aa1?gi=e0cf29a1474f)
2. **Liniar SVC**: [ learn more](https://pythonprogramming.net/linear-svc-example-scikit-learn-svm-python/#:~:text=The%20objective%20of%20a%20Linear,the%20%22predicted%22%20class%20is.)

Here is the code : 

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.cluster import KMeans
from sklearn.metrics import adjusted_rand_score
import json
import os
# Importing the libraries
import numpy as np
import pandas as pd
from flask import Flask,jsonify,request,render_template
app = Flask(__name__)


@app.route('/')
def main_page():
    return render_template('index.html')

@app.route('/kmeans', methods=['POST'])
def kmeans_pred():
    posted_data = request.get_json()
    true_k = posted_data['num_cluster']
    documents = posted_data['data']
    feeds = dict()
    vectorizer = TfidfVectorizer(stop_words='english')
    X = vectorizer.fit_transform(documents.split('.'))
    model = KMeans(n_clusters=true_k, init='k-means++', max_iter=100, n_init=1)
    model.fit(X)
    order_centroids = model.cluster_centers_.argsort()[:, ::-1]
    terms = vectorizer.get_feature_names()
    print(terms)
    for i in range(true_k):
        feeds.update({"cluster_"+str(i)+"": [terms[ind] for ind in order_centroids[i, :10]]})
    return jsonify(feeds)

@app.route("/ads", methods=['POST'])
def purshase():
    posted_data = request.get_json()
    age = int(posted_data['age'])
    salary = int(posted_data['salary'])
    new_data = [[age,salary]]
    # Importing the dataset
    dataset = pd.read_csv('Social_Network_Ads.csv')
    X = dataset.iloc[:, [2, 3]].values
    y = dataset.iloc[:, 4].values
    # Splitting the dataset into the Training set and Test set
    from sklearn.model_selection import train_test_split
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size = 0.25, random_state = 0)
    # Feature Scaling
    from sklearn.preprocessing import StandardScaler
    sc = StandardScaler()
    X_train = sc.fit_transform(X_train)
    X_test = sc.transform(X_test)
    new_data_pred = sc.transform(new_data)
    # Fitting SVM to the Training set
    from sklearn.svm import SVC
    classifier = SVC(kernel = 'linear', random_state = 0)
    classifier.fit(X_train, y_train)
    # Predicting the Test set results
    y_pred = classifier.predict(new_data_pred)
    response = {
        "result" : str(y_pred)
    }
    return jsonify(response)

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=os.environ.get('PORT'))

```

I will not explain the code, it's not the aim of this tutorial. But take a look a the last line :

```python
    app.run(host="0.0.0.0", port=os.environ.get('PORT'))
```

The app takes an environment variable called **PORT** ... but Why?

When you log in to your Heroku pannel you will create a simple server with a custom name, that name will be used later to access the app via the internet, in-depth it is a subdomain linked to your app, it's much like an nginx reverse proxy that will point the subdomain to you deployed app, How? 

Simply, When you hit the subdomain, Heroku will route your request to that port and return a response. If you choose your own port, your app won't work, because he doesn't know where to access the container.


# Ship the app

It's time to ship the app into an isolated container to be shippable on any Server!

```dockerfile
FROM frolvlad/alpine-python-machinelearning
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
EXPOSE 3000
ENV ENVIRONMENT dev
COPY . /app
CMD python main.py
``` 
This image `frolvlad/alpine-python-machinelearning` is ready to go image that contains what we need like the `sklearn` so we don't install it as a dependency, then we set our work directory to `/app` and copy the `requirements.txt` file to be used by the `pip` command to install `Flask`. By default, the app will run on port `3000` if we set the port on the `app.run(host='',port='')`. besides we need to copy the rest of the app code and run it with `python main.py`.   


# Test it locally 
To test locally follow these steps : 

```python 
# Clone the project 
git clone https://gitlab.com/hatemBT/REST-API-KMEANS
# Run it locally 
pip install -r requirements.txt
# Run the code 
python main.py
#To build the docker image (change the  port=os.environ.get('PORT') => port=3000 (you can choose yor custom port))
docker build -t kmeans_app . 
#Run with docker 
docker run -p 3000:3000 kmeans_app
```

Make sure to install python3 and docker engine.

# Prepare your Server

Log in to your Heroku account and then create a new app and name it `hello-devops-python` and choose the region from `US` or `Europe`, choose the nearest to you, hit create!

Notice that your app URL will be `hello-devops-python.herokuapp.com`.

We need the API key now, go to settings and grab that key. 

We need to create a special file called `.netrc` that will allow us to connect to Heroku `API` and it's `Git` :

```python
machine api.heroku.com
  login <mail>
  password <apiKey>
machine git.heroku.com
  login <mail>
  password <apiKey>
``` 

# Gitlab pipeline 

**Make sure to have a GitLab account and a repository**

We need to put the apiKey and the Heroku registry URL in GitLab as secrets.
go to  `settings->CI/CD->variables` and add `HEROKU_APP=registry.heroku.com/hello-devops-python/web`, note that `web` is the container name. and `HEROKU_TOKEN` that hols the API key.

![Screenshot_2020-11-28 CI CD Settings · CI CD · hatem ben tayeb REST-API-KMEANS.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1610477581881/uIQKyhTEy.png)


The pipeline has a single-stage `development` that will deploy the container to the "hello-devops-python" app. Let's write the CI/CD configuration (.gitlab-ci.yml) : 

```yaml
image : docker:latest
services:
  - docker:dind
variables:
    DOCKER_DRIVER: overlay
stages:
    - development
devloppement:
    stage : development
    script:
         - docker build -t $HEROKU_APP .
         - docker login --username=_ -p $HEROKU_TOKEN registry.heroku.com
         - docker push $HEROKU_APP
         - docker run  --rm  -e HEROKU_API_KEY=$HEROKU_TOKEN wingrunr21/alpine-heroku-cli container:release web --app hello-devops-python
```
Simply this pipeline follows these steps :


- **Build the container**: `docker build -t $HEROKU_APP .`
- **login to Heroku registry**: `docker login --username=_ -p $HEROKU_TOKEN registry.heroku.com`
- **Push the container**: `docker push $HEROKU_APP`
- **Release the container**: `docker run  --rm  -e HEROKU_API_KEY=$HEROKU_TOKEN wingrunr21/alpine-heroku-cli container:release web --app hello-devops-python`, we used a temporary docker image that contains the `heroku-cli` pre-installed, to deploy the `web` container to `hello-devops-python`.

**Push the code**
![giphy.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1610479086446/ZClr9_XxO.gif)
**Wait unit Green**

![Screenshot_2020-11-28 Pipelines · hatem ben tayeb REST-API-KMEANS.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1610479158601/iCve4rXPC.png)

# Test The App

You can access your app now via the browser, let's test the app API :

Test the **/kmeans** route :
```bash
curl -X POST -H "Content-Type: application/json" "http://hello-devops-python.herokuapp.com/kmeans" -d @data.json | jq "."
```
Example data : 
```json
{
    "num_cluster":2,
    "data":"
The brothers wanted to build their own kingdom where people could live without fear.
 They collected a band of young men and trained them in warfare
 They lived in a forest hideout on the banks of the river Tungabhadra in South India.
One day, the brothers were out on a hunt. Ferocious dogs accompanied them.
 They crossed the river and rode on 
A couple of frightened rabbits ran out of the bushes
The dogs gave them chase with the two brothers closely behind on their horses.
"
}
```

Output: 
```json
{
  "cluster_0": [
    "accompanied",
    "ferocious",
    "dogs",
    "gave",
    "forest",
    "fear",
    "day",
    "crossed",
    "couple",
    "collected"
  ],
  "cluster_1": [
    "brothers",
    "hunt",
    "day",
    "river",
    "wanted",
    "fear",
    "people",
    "build",
    "live",
    "kingdom"
  ]
}
```
Test the **/ads** route :
```bash
curl -X POST -H "Content-Type: application/json" "http://hello-devops-python.herokuapp.com/ads" -d "{\"age\":80,\"salary\":190000}" | jq "."
```
Output : 

```json
{
  "result": "[1]"
}

```

The presentation version :

%[https://slides.com/hatemtayeb/deck-a805ec]