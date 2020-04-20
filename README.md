# SAP Data Intelligence (SAP DI)

[TOC]

## Building a  simple pipeline to train a ML Model

The following is based on the **[SAP Data Intelligence: Create your first ML Scenario](https://blogs.sap.com/2019/08/14/sap-data-intelligence-create-your-first-ml-scenario/)** by [Andreas Forster](https://people.sap.com/andreas.forster). This guide includes updates based on changes made by SAP and some more in depth content on the settings for the operators.

After the exploratory part (data exploration & freestyle data science) in Jupyter Labs, the final developments can be delivered via pipelines. A simple pipeline can be put together from just a a few steps and in this case is exemplarily constructed as follows[^1]:

<img src="F:\OneDrive - Verovis GmbH\Projekte\SAP Data Intelligence\Marathon_Times_Tutorial\Bilder\2020-04-16 15_17_41-Modeler _ SAP Data Intelligence.png" alt="2020-04-16 15_17_41-Modeler _ SAP Data Intelligence"  />



### Prerequisites

The file for the machine learning part can be found [here](https://raw.githubusercontent.com/AndreasForster/Data-Intelligence/master/Create%20your%20first%20ML%20Scenario/RunningTimes.csv).

For creating a pipeline navigate to the ML Scenario Manager and click on the **+** icon on the top-right hand:

![](2020-04-20 14_54_07-ML Scenario Manager.png)

In the newly created ML Scenario navigate to the **Pipelines** section and click on the **+** icon on the top-right hand:

![](2020-04-20 15_03_42-ML Scenario Manager.png)

Enter a **name** and select the **blank** template:

![](2020-04-20 15_07_44-ML Scenario Manager.png)

### Operator Settings

#### Read File

- [ ] Read: Set to **once** 

- [ ] Connection: Chose the connection via the **connection management** drop down menu. To access the DI Data lake, the settings are as follows

<img src="F:\OneDrive - Verovis GmbH\Projekte\SAP Data Intelligence\Marathon_Times_Tutorial\Bilder\2020-04-16 16_01_19-Modeler _ SAP Data Intelligence.png" alt="2020-04-16 16_01_19-Modeler _ SAP Data Intelligence" style="zoom:67%;" />

- [ ] Path: contains the path to the file, either give a direct path or use `${inputFilePath}` to be able to provide the path and file when executing the pipeline

#### ToString Converter

* Preconfigured and no actions needed

#### Python3

On the left hand menu bar, click on `operators` and select the `Python3 operator`. Drag and drop it to the canvas. The operator comes without **input** or **output ports**, thus they have to be added to the *Python3 operator*. The *python3 operator* is used to execute our python script which trains the model, calculates the metrics and sends the outputs to the output ports.

##### Port Settings

The pipeline uses **one** input file for the data input that is leveraged in our python3 operator to train the model. The python operator generates **two** outputs (metrics and model) for further use in the pipeline. 

To add ports click on the python operator and select the **Add Port** icon

<img src="F:\OneDrive - Verovis GmbH\Projekte\SAP Data Intelligence\Marathon_Times_Tutorial\Bilder\2020-04-16 16_08_50-Modeler _ SAP Data Intelligence.png" alt="2020-04-16 16_08_50-Modeler _ SAP Data Intelligence" style="zoom:67%;" />

- [ ] The settings for the input port are as follows

<img src="F:\OneDrive - Verovis GmbH\Projekte\SAP Data Intelligence\Marathon_Times_Tutorial\Bilder\2020-04-16 16_10_29-Modeler _ SAP Data Intelligence.png" alt="2020-04-16 16_10_29-Modeler _ SAP Data Intelligence" style="zoom:67%;" />

- [ ] The settings for the output port are as follows

<img src="F:\OneDrive - Verovis GmbH\Projekte\SAP Data Intelligence\Marathon_Times_Tutorial\Bilder\2020-04-16 16_10_52-Modeler _ SAP Data Intelligence.png" alt="2020-04-16 16_10_52-Modeler _ SAP Data Intelligence" style="zoom:67%;" />

<img src="F:\OneDrive - Verovis GmbH\Projekte\SAP Data Intelligence\Marathon_Times_Tutorial\Bilder\2020-04-16 16_11_07-Modeler _ SAP Data Intelligence.png" alt="2020-04-16 16_11_07-Modeler _ SAP Data Intelligence" style="zoom:67%;" />

##### Code (Script) Settings

Access the script settings of the *Python3 operator* 

<img src="F:\OneDrive - Verovis GmbH\Projekte\SAP Data Intelligence\Marathon_Times_Tutorial\Bilder\2020-04-20 10_23_32-Window.png" alt="2020-04-20 10_23_32-Window" style="zoom:67%;" />

and replace the exemplary  code with the following:

```python
# Example Python script to perform training on input data & generate Metrics & Model Blob
def on_input(data):
    
    # Obtain data
    import pandas as pd
    import io
    import pickle
    df_data = pd.read_csv(io.StringIO(data), sep=";")
    
    # Get predictor and target
    x = df_data[["HALFMARATHON_MINUTES"]]
    y_true = df_data["MARATHON_MINUTES"]
    
    # Train regression
    from sklearn.linear_model import LinearRegression
    lm = LinearRegression()
    lm.fit(x, y_true)
    
    # Model quality
    import numpy as np
    y_pred = lm.predict(x)
    mse = mean_squared_error(y_true, y_pred)
    rmse = round(np.sqrt(mse), 2)
    
    # to send metrics to the Submit Metrics operator, create a Python dictionary of key-value pairs
    metrics_dict = {"RMSE": str(rmse), "n": str(len(df_data))}
    
    # send the metrics to the output port - Submit Metrics operator will use this to persist the metrics 
    api.send("metrics", api.Message(metrics_dict))

    # create & send the model blob to the output port - Artifact Producer operator will use this to persist the model and create an artifact ID
    model_blob = pickle.dumps(lm)
    api.send("modelBlob", model_blob)

api.set_port_callback("input", on_input)
```

The *Python3 operator* receives the data from the *read file operator* and then executes the code to train the model. The model metrics are calculated, saved as a dictionary and then send to the *“metrics” output port* from where they get submitted by the *Submit Metrics operator* to the ML Scenario. The ML model is saved as pickle file and then send to the *“modelBlob” output port* where the *Artifact Producer operator* consumes the ML model, persists it and creates an artifact ID which is also passed to the ML Scenario.

#### Submit Metrics

- Preconfigured and no actions needed

#### Artifact Producer

- Preconfigured and no actions needed

#### ToMessage Converter

* Preconfigured and no actions needed

#### ToString Converter

* Preconfigured and no actions needed

#### Operators Complete

To check if all actions were carried out successfully and terminate the graph afterwards select the *Python3 operator* from the operators menu and drag and drop it to the canvas. The procedure needs two input (metrics & ML model) and one output (completed message) port. The ports can be added in the same way again (see [Port Settings](#port-settings)). The settings for the ports are as follows: 

- [ ] Input

<img src="2020-04-20 11_12_41-Modeler _ SAP Data Intelligence.png" style="zoom:67%;" />

<img src="2020-04-20 11_13_02-Modeler _ SAP Data Intelligence.png" style="zoom:67%;" />

- [ ] Output

<img src="2020-04-20 11_13_13-Modeler _ SAP Data Intelligence.png" style="zoom:67%;" />

#### Graph Terminator

* Preconfigured and no actions needed

### Dockerfile

To run the *Python3 operator* it is necessary to provide a dockerfile which builds the prerequisites for the pipeline (like a virtual environment). To build a docker file, proceed as follows:

- [ ] Click on `Repository`
- [ ] Click on `dockerfiles`
- [ ] Click on `Create docker file`

<img src="2020-04-20 11_23_31-Modeler _ SAP Data Intelligence.png" style="zoom:67%;" />

Name the dockerfile `python36marathon`and paste this code into the docker file code windows: 

```python
FROM $com.sap.sles.base

RUN pip3.6 install --user numpy==1.18.3
RUN pip3.6 install --user pandas==1.0.3
RUN pip3.6 install --user scikit-learn==0.22.2
```

By calling `FROM $com.sap.sles.base` it is possible to leverage a base image and build a custom dockerfile on top. It’s advised to specify the  versions of the libraries to ensure that new versions of these libraries do not impact your environment.

SAP DI uses tags associated to the docker files to indicate the content of each docker file. Furthermore a custom tag is used to specify which docker image is used to run an operator. Open the configuration panel for the docker file with the icon on the top-right hand corner.

SAP Data Intelligence uses tags to indicate the content of the Docker  File. These tags are used in a graphical pipeline to specify in which  Docker image an Operator should run. Open the Configuration panel for  the dockerfile with the icon on the top-right hand corner.

![](2020-04-20 13_22_15-Window-1587381911114.png)

At runtime, the new dockerfile tags are automatically expanded to {"sles": "", python36": "", "tornado": "5.0.2", "numpy36": ""} due to the automatic tag inheritance mechanism (the parent dockerfile com.sap.sles.base contains the tags {"sles": "", python36": "", "tornado": "5.0.2"}).[^2]

Adding custom tags to the dockerfile helps to identity the files used for specific settings, i.e. `marathontimes` for this pipeline. To indicate that the dockerfile ships with scikit-learn and numpy, add the corresponding tags to the file (either use the existing ones or generate custom tags on your own and add the version).

![](2020-04-20 13_51_00-Window.png)

Afterwards save and build the docker file:

![](2020-04-20 13_55_49-Window-1587383894636.png)

After a few minutes the successful build should be confirmed:

![](2020-04-20 14_12_53-Window.png)

The dockerfile has to be attached to the operations that should be carried out based on the dockerfile configuration. Click on the *Python3 operator* and select *group*.

 ![](2020-04-20 14_22_28-Modeler _ SAP Data Intelligence.png)

Afterwards, select the group and add the custom tag *marathontimes* to the group to link the group to the marathontimes dockerfile.

![](2020-04-20 14_24_37-Window.png)

![](2020-04-20 14_26_17-Window.png)

Afterwards save the graph by clicking on **save** on the top-middle menu bar:

![](2020-04-20 15_11_42-Modeler _ SAP Data Intelligence.png)

### Test Run and Versioning

To test the pipeline it is possible to execute it from the *DI Modeler* by clicking on **run** on the top-middle menu bar: 

![](2020-04-20 14_31_01-Modeler _ SAP Data Intelligence.png)

The complete ML Scenario can be accessed in the ML Scenario Manager. Create a new version of the ML Scenario by clicking on **Create Version** on the top-right hand. The Scenario overview also shows the **Progress Flow** of the pipeline:

![](2020-04-20 14_42_02-ML Scenario Manager.png)

as well as the calculated metrics:

![](2020-04-20 14_42_45-ML Scenario Manager.png)

and the persisted ML model: 

![](2020-04-20 14_46_24-ML Scenario Manager.png)

## Expanding the simple Model

### Reading more than one input file

- [x] Mehr als ein File einlesen
- [x] Mehr als ein File nutzen
  - [x] Preprocessing
  - [x] Model

### Writing output into Hana DB

- [x] Output in eine eigene Hana Tabelle schreiben (vorher erstellt)
- [ ] Output in eine eigene Hana Tabelle schreiben (Hana Tabelle in der Pipeline anlegen)
- [ ] Output in eine vorhandene Handa Tabelle schreiben (join)

## Useful links

https://blogs.sap.com/2019/08/14/sap-data-intelligence-create-your-first-ml-scenario/

https://blogs.sap.com/2019/12/16/orchestrierung-von-hana-sdi-mit-sap-data-intelligence/

https://blogs.sap.com/2019/12/15/how-to-use-the-hana-ml-library-within-a-python-operator-in-sap-data-intelligence/

https://blogs.sap.com/2019/11/05/hands-on-tutorial-machine-learning-push-down-to-sap-hana-with-python/

https://blogs.sap.com/2020/04/15/creating-hana-cloud-foundry-connection-with-sap-data-intelligence-and-applying-random-forest/

https://blogs.sap.com/2018/01/15/sap-data-hub-develop-a-custom-pipeline-operator-from-a-base-operator-part-1/

https://blogs.sap.com/2018/01/15/sap-data-hub-develop-run-monitor-and-trace-a-data-pipeline-part-2/

https://blogs.sap.com/2018/01/23/sap-data-hub-develop-a-custom-pipeline-operator-with-own-dockerfile-part-3/



[^1]: The original content includes also the data exploration and freestyle data science parts. This guide includes updates and some more in depth content on the settings for the operators. The original content is available here: https://blogs.sap.com/2019/08/14/sap-data-intelligence-create-your-first-ml-scenario/#freestyle
[^2]: More information regarding docker inheritance: https://help.sap.com/viewer/29ff74dc606c41acad117003f6034ac7/2.7.latest/en-US/d49a07c5d66c413ab14731adcfc4f6dd.html
