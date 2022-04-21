# Home assignment: build your credit model

In this assignment, you will analyze an artificial dataset, and build a credit model
to predict future unpayments.

We provide you with a training dataset, in which you will be able to investigate
the history of loan requests (orders) of a set of customers, as well as when they paid.
In the test dataset, we provide the details of the last transaction requested by
some of these users, and ask you to estimate the likelihood of the customers not paying
it.

## Installation instructions

In this folder, we provide a Docker setup, which contains a postgres database image, and a
jupyter-lab image. You will need to have a recent version of docker compose installed.
Once done, you can simply run:

```bash
# Make database connection parameters accessible to docker
cp .env.template .env
# Launch the services in the image. Addd `--renew-anon-volumes` to empty the database
docker-compose up
```

The first time, you will additionally need to load the data:

```bash
source .env && docker-compose exec -T db psql -U${POSTGRES_USER} ${POSTGRES_DB} < data/dump.sql
```

You can access the Jupyter Lab instance from docker compose in your browser, by browsing to:
`localhost:10000`. You can find the security token in the terminal where you launched
`docker-compose up`.

The folder `work` is mapped in docker, and it contains the notebook `sample-notebook.ipynb`,
with instructions on how to interact with the database.

## Dataset information

The dataset is provided as a SQL dump (see the installation instructions). It contains the
following datasets:

* **buyers**: buyers to be investigated. It contains the following columns:
  * `buyer_id`: unique buyer identifier
  * `first_name`
  * `last_name`
  * `email`
  * `created_at`: timestamp of the first interaction with this buyer
* **merchants**: merchant selling the goods. It contains the following columns:
  * `merchant_id`: uniquer merchant identifier
  * `name`
  * `created_at`: timestamp of the first interaction with this merchant
* **orders_creation**: order creation events. It contains the following columns:
  * `order_id`: unique order identifier
  * `buyer_id`: buyer in this order
  * `merchant_id`: merchant in this order
  * `total_price_cents`: basket size (in cents of euro)
  * `created_at`: timestamp when the event occurs
* **orders_decision**: event with the decision whether to accept or reject an order.
  It contains the following columns:
  * `order_id`: unique order identifier
  * `decision`: decision taken (either `ACCEPT` or `REJECT`)
  * `due_date`: payment due date
  * `created_at`: timestamp when the event occurs
* **orders_payment**: payment event
  * `order_id`: unique order identifier
  * `created_at`: timestamp when the event occurs
* **datasets**: label for the different orders:
  * `order_id`: unique order identifier
  * `dataset_type`: dataset type (either `train` or `test`)

## Process

During this exercise, you will need to build a simple credit model. **Your model will be called when a
new order is created**, and must help in the decision to accept or reject it. For that, you will estimate
the probability of the order not being paid by the buyer, 5 days after the due date. 

The dataset does not contain many features, so an important part of your task will be to create your own
features. As a hint, please notice that all the orders in the test dataset correspond to
**returning customers**, i.e., we have previous information on that particular customer. The steps required
to successfully complete this task are described below.

### 1: create your raw train and test datasets

Download the data and combine the events into a single pandas dataset with all the relevant features.
Keep in mind that your model will be called when the order is called, so you cannot use information after that
point in time. Also, be careful about not having data leakage between training and testing!

### 2: create your target

As mentioned above, our target for an order is it not being paid by the buyer, 5 days after the due date.

### 3: explore different features, and build simple models based on them

Use the history available to define relevant features. As an example, you may want to define the feature
`Was the last order more expensive than the current one?`. Do not go for complicated features, as you will
later need to implement them in SQL!

Then, use a simple model (logistic regression, gradient boosting, etc.) to do proper classification.

### 4: fit a model, recalibrate it, and evaluate it in your dataset splits

This exercise is not about having a great model, but rather about building meaningful features and understanding
the model performance. Also, your deliverable must be real probabilities, so remember to recalibrate it after
fitting!

### 5: build your features in SQL, and make sure that they are correct

When we deploy a model, we typically cannot load the full history of transactions, every time we need to make
inference on an observation. A possible approach is to define a SQL query to retrieve the value of that feature.
Below you can find an example on how to get the query for the feature
`Was the last order more expensive than the current one?`:

```python
def buyer_last_order_is_more_expensive__query(buyer_id: int, merchant_id: int, total_price_cents: int, created_at: str) -> str:
    return f"""
    WITH source AS (
        SELECT
            *,
            ROW_NUMBER() OVER (PARTITION BY buyer_id ORDER BY created_AT DESC) AS order_recency
        FROM orders_creation
        WHERE
            buyer_id = {buyer_id}
            AND created_at < '{created_at}'::TIMESTAMP
    )
    SELECT total_price_cents > {total_price_cents} AS buyer_last_order_is_more_expensive
    FROM source
    WHERE order_recency = 1
    """
```

Can you do that for the features your model is using?
