# Getting started with DaedalusML

DaedalusML provides a simple interface for setting up an ML training pipeline.  It exclusively relies on HTTP endpoints as a data source.  While the guide provided here is primarily for those users who do not have a publicly exposed API to use as a data source, it contains useful information about the app for all users.  

For this walkthrough, I am using Windows with a desktop resolution.  Steps may differ by operating system or device.

## Contents



## 1. Install Prerequisites
For this demo, we will be using the following tools:

* Python 3: available [here](https://www.python.org/)
* VSCode with support for Jupyter notebooks: available [here](https://code.visualstudio.com/). There's a handy how-to guide to help you set up Jupyter notebook support [here](https://code.visualstudio.com/).
* .Net 7 SDK: available [here](https://dotnet.microsoft.com/en-us/download/visual-studio-sdks?cid=getdotnetsdk).
* Ngrok: a handy tunneling and ingress-as-a-service tool available [here](https://ngrok.com/).  If you only want to follow along without incurring any costs, be sure to sign up for the free plan they have available.
*  (Optional) SQLiteStudio: a free, cross-platform SQLite database management utility, available [here](https://sqlitestudio.pl/).

## 2. Clone the getting-started repository

* Open a terminal.
* Browse to the destination directory you would like to clone to.
* Copy & paste the following to clone the repository:

```
git clone https://github.com/BackOfficeIntegrations/getting-started.git .
```

## 3. Run the Jupyter notebook

From the same terminal, type the following to open the working directory in vscode:

```
code .
```

Select the `getting-started.ipynb` file.  If you've set up python & vscode correctly, you should be able to run each cell contained in the notebook in order to produce the correct result.  Note: you may have to install additional packages for python as detailed in the notebook.

## 4. (Optional) Inspect the database file with a database management tool

There are a number of free, open-source options available for working with SQLite3.  SQLiteStudio is a free and useful choice.  A link is provided in the prerequisites list at the beginning of this guide.  

Once you have installed your database management tool, browse the newly created `.db` file located in the working directory used for the Jupyter notebook to inspect its contents. 

## 5. Install and run the serve-sqlite DotNet global tool

In your terminal, type:
```
dotnet --version
```
If the .Net SDK has been installed corretly, the response should report version 7.x.x.

Next, type the following into your terminal to install the tool:
```
dotnet tool install --global serve-sqlite --version 1.0.1
```
Once installed, the tool will be available as the command `serve-sqlite`.  From the working directory of your Jupyter notebook, you can now start an api for the database with the following:
```
serve-sqlite -p getting-started.db --https 443
```
You should now be able to point your browser to `https://localhost:443/xor` to view data served by the api. Note: `serve-sqlite` supports OData query strings, which is a useful feature for supporting paging, filtering, sorting and counting.  For more information, OData's [tutorial](https://www.odata.org/getting-started/basic-tutorial/#queryData) is a good starting point.

## 6. Create a public api using Ngrok

Open a new terminal.  If installed correctly, `ngrok` should be an available command.  Type the following to test that it works:
```
ngrok --version
```
Next, the following command will create a public api which points to your local api:
```
ngrok http https://localhost:443
```
If you're using the free plan, ngrok will generate a pseudo-random url for your public endpoint.  It can be copied from the display:

![Ngrok display](/images/ngrok.png)

## 7. Create a new ET pipeline

* Open the DaedalusML web application.
* In the upper-left quadrant, under ET Pipelines, click `New` to create a new ET pipeline.
* Enter the name `getting-started` & a batch count of 10.
* The page for the new ET pipeline should now open.  In the first cell, enter the following code:

```
var url = 'https://localhost:443/xor';
var pageSize = 4;
var skip = $$.batchIndex * pageSize;
var query = {
	$skip: skip,
	$top: pageSize
};
var batch = $$.httpGet(url, { query });
var df = $$.dataFrame(batch);
df
```
If everything is set up correctly, after running the cell you should see the following output (click on `Apply Formatting` if needed):

![Cell 1 output](/images/cell-1-output.png)

There are a few details to note here:
* Scripts defined on the ET pipeline page are executed once per batch.  
* The global toolbox symbol (`$$`) is a special designator for the global toolbox which contains special functions and values.
* `$$.batchIndex` will always contain the zero-based batch index for the current run.
* Http functionality is currently limited to GET requests, accessible via the `$$.httpGet` function, which takes two parameters: firstly the url, and secondly an optional options object defined as `{ headers, params }`.  Both `headers` and `params` are used as simple key-value-pair dictionaries.
* For the `$$.dataFrame` function, the app uses dataframe-js.  You can read more about the functionality it provides [here](https://github.com/Gmousse/dataframe-js).
* The final line in this code block pushes a dataframe into a global results collection. The result of each cell (if one exists) is accessible in the following cell using `$$.input`. 

For the next step, we'll continue the extraction process by transforming the data.  In the second cell, enter the following:

```
var x = df.select('Input1', 'Input2').toArray();
var y = df.select('Output').toArray();
$$.output(x, y)
```
Note:
  * The `$$.output` function combines & transforms array inputs into the shape that is required by the app's training algorithms.
  * The output of this function is left here on the last line of the ET pipeline script to designate it as the final value pushed into the global results collection of the extraction & transformation process.

So far, you've been running and testing this code locally, but you will need to upload it to the cloud for training.  Referencing local host from within the cloud will not work, which is why you will need to use an ingress service like Ngrok if you're hosting the data from your local machine.  Replace the localhost portion of the url in the first cell's script with the url you obtained from the Ngrok agent, but retain the `xor` route.

Lastly, in this scenario you shouldn't need to worry about rate limitations on your api, but in the case you rely on an API that does impose limitations, from the ET pipeline page you can configure rate limits in the setting menu accessible from the upper-right corner of the section.

## 8. Create a model definition

Once you're done configuring your ET pipeline, click on the `Overiew` link in the upper-left corner of the screen.

In the lower-left quadrant, select `New`.

A modal should appear allowing you to configure your model definition.  Configure as follows:

![Model configuration 1](/images/model-configuration.PNG)

Click on `Edit` under `Layers`.  Configure as follows:

![Model configuration 2](/images/model-configuration-2.PNG)

Then, submit your changes.

## 9. Create an ML pipeline

In order to create an ML pipeline, you first need to have an available and compatible ET pipeline & model pair.  If you've been following the steps to this point, you should be all set.  From the Overview page, first select your ET pipeline and then your model.  Each should be highlighted once you have.  Now, in the upper-right quadrant under ML Pipelines, select `New`.

Configure the the pipeline as follows and submit your changes:

![ML pipeline configuration](/images/ml-pipeline-configure.PNG)

## 10. Train a model

You're all set!  After selecting the ml pipeline from the Overview page, you should see an `Execute Pipeline` option appear in the lower-right quadrant.  

## 11. Download a trained model

If your training run completes successfully, here's how you can access download options:
* Hover over the model in the lower-left quadrant.
* Click on the "slide left" icon.
* A menu should appear in line with the model.  Select `View`.
* In the training result modal that opens, you will be presented with a variety of options.
