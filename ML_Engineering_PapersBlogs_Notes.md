## Booking:

#### Machine Learning in production: the Booking.com approach 
	
- Link: https://booking.ai/https-booking-ai-machine-learning-production-3ee8fe943c70	
- Description: This blogpost from booking.com is about RS, their in-house framework to support productization of ML models.
- Technologies: RS - proprietary frame to run ML models into prod.
- More: The most important requirement to be met by models in production are:
    - Consistency: the online predictions must match the offline predictions and they must be the same regardless of which Data Center, Server, Pod, etc. is hit by the request.
    - High availability: 24/7
    - Low latency: the model must make predictions really fast. Many models collaborate to achive a goal, so even if individual models are not slow, the aggregated latency might become prohibitively high.
    - Scalability: the models must be ready to take in growing number of requests per second.
    - Observability: the models have to be observed to react appropriately and promptly.
    - Reusability: many of our models solve a rather generic task. This model can be applied in many different product features. We can use it to highlight those family-friendly hotels in the details page or to create a filter in the search results page, or to reinforce a user decision in the booking process. These are all examples of model reutilization.
 
 RS platform has four basic mechanisms to productionize machine-learned models:
    - *Lookup tables*: Most of the models map an input vector to a prediction (or set of predictions in the case of recommender systems). A very simple way to deploy a model in production is to precompute all the predictions for all the possible inputs and store them in a key-value store. At prediction time all we need to do is lookup the prediction using the input as the key. Advantages:
        a) latency is usually very low due to absence of computation at prediction time and since most key-value stores are optimized for fast reading, 
        b) horizontal scalability is quite straightforward, most key-value stores take care of it.
        c) huge modeling flexibility. It doesn’t really matter how the model was trained, as long as the predictions can be computed and written to the key-value store in time the model can be productionized reliably.
    It also has several drawbacks:
        i) When the input space is large, it might be difficult to store that many combinations, or impossible to precompute all the predictions within a reasonable time. Besides, chances are many of the input combinations will never actually happen in production, resulting in waste of computation and storage resources.
        ii) The process of writing the predictions to the key-value store might be cumbersome when we consider model versions, modifications to the feature space, etc.
        iii) Continuous inputs are not supported.
    - Generalized Linear Models (GLMs)*: In this method, the model is represented by a scalar weight for each input plus a global bias; at prediction time, we just compute the inner product of the input with the weights, add the bias, apply a scalar function (the “inverted link”) and return. In general, this simple prediction rule allows serving predictions for many kinds of regressors, binary or multiclass classifiers, recommender systems or rankers etc. It doesn’t matter which training algorithm is used as long as it can be represented by a weight vector, an input transformation, and link function. Due to its flexibility and online performance, this method is very popular and applied to deploy many models in a variety of business cases like user preference models, user context models, destination recommendations, budget prediction, hotel attributes prediction and more.
    - *Native libraries*: This method consists of simply using the library used to train the model to make predictions in production. This method brings a new dimension into the picture: ease of use. We can simply train our model and upload it without any transformation to intermediate formats. It also provides high consistency, the same code used to train is used to predict, no surprises in production. On the other hand, native libraries might be difficult to deploy, since they require a specific runtime environment. Another disadvantage is that native libraries might not be optimized for the serving time but rather for training time, increasing the risk of latency.
    - *Scripted models*: A scripted model is simply a script with a predefined interface that is invoked for every request. This approach gives huge flexibility since it allows, among other things, to control the run-time environment, and to perform complex tasks at prediction time. On the other hand, scripted models are a weak link in the online request life cycle: every line of code will have an impact at prediction time, increasing the risk of latency and failure.

These are the four canonical methods that allow us to make sensible trade-offs. Not all business cases require the same level of modeling flexibility or robustness, and for one single business case, these requirements are also not at the same level on different stages of the solution. Flexibility and Robustness are two software attributes creating a trade-off plane: the more Flexibility a method offers, the less Robust it is and vice-versa. Flexibility can be decomposed into flexibility with respect to the Input Space, to the Modelling Approach or to the Stack (programming language and libraries). Likewise, Robustness can be decomposed into Latency, Consistency (between the training model and the actual running artifact), and Observability.

The main idea behind RS is to decouple training from prediction. It doesn’t matter how a model was built, or which productionization method is used, model consumers can use exactly the same API. Obtaining
prediction is just one functionality, many others can also be decoupled from training and productionization method, like monitoring and A/B testing. Models can be uploaded through a programmatic interface or through a web portal. RS distributes the model across nodes in a cluster where a Java process takes care of loading them into memory and makes them available to serve predictions through a standard HTTP interface. One RS node serves many models and any given model is loaded in many nodes, this approach helps to achieve High Availability and Scalability requirements.
Lookup tables are implemented using the Cassandra key-value store (or in memory if they are small enough). GLMs are served through an in-house developed linear prediction system that uses simple text files as model descriptors. Native Libraries are supported through H2O MOJOS, Tensorflow, and Vowpal Wabbit binaries. Scripted models are Python scripts, each running in its virtual environment allowing users to upload additional modules and dependencies as needed. On top of this, RS adds a lot of value through extra functionality, including a web portal that allows to search and browse all the available models. Each model has its own page with detailed information such as experiments using the model, monitoring tools, documentation, link to the training code, etc. It also provides a basic state machine that allows the author to transition the model through states like in-testing, production-ready, disabled, etc.