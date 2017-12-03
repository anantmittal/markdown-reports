> Independent Study Research with Professor Christopher Brooks
``` 
Anant Mittal, School of Information, University of Michigan
anmittal@umich.edu
Date: 3rd December 2017
```
## Introduction

![1](https://github.com/anantmittal/markdown-reports/blob/master/images/Untitled.png) 



Professor Christopher Brooks runs a set of massive open online courses (MOOCs) under Applied  Data  Science  with  Python  Specialization on Coursera. The weekly assignments for the courses are on Jupyter Notebooks where the students need to solve the questions posted in each cell.

Above is a template of the weekly assignment. The first few cells load the dataset in a dataframe after which there are different questions revolving around the dataset. The students need to submit their answers in the form of a python definition in the cell below each question. 

## Logging Extension 

### What is it?

When a student executes the code in jupyter notebook, the content of the notebook gets saved in AWS DynamoDB in the following format. 

```
{
  "cell_data": {
    "cell_metadata": {
      "trusted": true
    },
    "code": "x = np.random.binomial(20, 0.5, 10)\nx"
  },
  "expire": 1512326556,
  "kernel_output": {
    "buffers": [],
    "channel": "iopub",
    "content": {
      "ename": "NameError",
      "evalue": "name 'np' is not defined",
      "traceback": [
        ]
    },
    "header": {
      "date": "2017-12-01T18:42:36.071580",
      "msg_id": "032b34ff-a3be-4d32-b1a9-7ccb29804f63",
      "msg_type": "error",
      "session": "356b6142-e5eb-428e-80a4-7131757fdecd",
      "username": "username",
      "version": "5.0"
    },
    "metadata": {},
    "msg_id": "032b34ff-a3be-4d32-b1a9-7ccb29804f63",
    "msg_type": "error",
    "parent_header": {
      "date": "2017-12-01T18:42:35.902289",
      "msg_id": "FD09CD3C33CC4908A2E6A22D39BCCAFE",
      "msg_type": "execute_request",
      "session": "69563C9C4A344E0BB51B794D03E74E87",
      "username": "username",
      "version": "5.0"
    }
  },
  "notebook_metadata": {
    "kernelspec": {
      "display_name": "Python 3",
      "language": "python",
      "name": "python3"
    },
    "language_info": {
      "codemirror_mode": {
        "name": "ipython",
        "version": 3
      },
      "file_extension": ".py",
      "mimetype": "text/x-python",
      "name": "python",
      "nbconvert_exporter": "python",
      "pygments_lexer": "ipython3",
      "version": "3.6.0"
    }
  },
  "timestamp": 1512153756,
  "url": "https://hub.coursera-notebooks.org/user/afwoejlriewvezcumpbedw/notebooks/Week%204.ipynb",
  "user_id": "afwoejlriewvezcumpbedw",
  "version_date": "11/09/2017"
}
```

The keys in the above json are explained below:
* cell_data 
  * cell_metadata - Contains the metadata of the cell which has been executed
  * code - Contains the python code in the cell
* expire - holds the timestamp after which the whole json should be removed from the AWS DynamoDB (currently being initialized to 2 days after insertion in DynamoDB)
* kernel_output - Contains the json which is returned by the IPython kernel after executing the python code in the cell. Holds the output, error messages, etc. 
* notebook_metadata - Contains the metadata of the whole notebook 
* timestamp - holds the server timestamp when the row was inserted in DynamoDB, or in other terms when the cell code was executed
* url - the coursera url 
* user_id - the coursera user id
* version_date - the date of when the format of json was last updated

### How does it work?
Typically, when you execute a cell in jupyter notebook, the client side of jupyter strips the metadata off the notebook json and sends just the python code to IPython kernel for execution. Once the client side gets back the output from the kernel, it renders it back on the jupyter notebook. 

#### 1st lifecycle of the extension

We developed one jupyter notebook front-end extension and created a new kernel entirely which extended the original IPython kernel. 

| Client/Server | Method Overridden | Original Functionality | New functionality |
| --- | --- | --- | --- |
| Client | Jupyter.CodeCell.prototype.execute | Send the python code to IPython kernel and render the results received | Append the cell metadata, notebook metadata, coursera user id, etc. to the python code, send it to IPython kernel and render the results received |
| Server | do_execute | Receive the python code, execute it and return the output to the client-side | <ul><li>Receive the new json, strip off the python code, execute it and return the output to the client-side</li><li>Send the new json to a queue which would HTTP POST the json to a server side AWS Lambda instance built using Flask and Python</li><li>The goal of this lambda instance is to save the whole json in AWS DynamoDB</li></ul>|


The motivation behind the this architecture was to test if we can move some part of the computation to Coursera'server. Additionally, we wanted to check the range of possibilities, of how much we could change the coursera's IPython kernel code for raising future research questions. 

Once we deployed this architecture on Courser'a platform, the front-end extension worked out well. But, the server side of IPython kernel failed to make HTTP POST calls to our AWS Lambda instance. For security concerns, Coursera's team had setup a firewall because of which our kernel extension was breaking. This lead us to the second phase of logging extension. 

_Note: This phase did not use to save the "kernel_output" json in AWS DynamoDB_

#### 2nd lifecycle of the extension

| Client/Server | Method Overridden | Original Functionality | New functionality |
| --- | --- | --- | --- |
| Client | Jupyter.CodeCell.prototype.execute | Send the python code to IPython kernel and render the results received | Create a new json with cell code, cell metadata, notebook metadata, coursera user id, etc. |
| Client | Jupyter.CodeCell.prototype.get_callbacks | Would get called as a callback once the above execute method would get completed. Responsible for rendering the output. | Will append the output received from the IPython kernel to the new json and send it to the method which would HTTP POST the new json to the AWS Lambda instance|

_Note: The "kernel_output" was added in the 2nd phase. This phase did not involve extending the IPython kernel._   

## Analysis

To start thinking about the kind of analysis we could perform, we came across code clone detection tools which are used to detect redundancy in the code and in turn reduce the maintenance cost. Although, the use case for such tools is different, we tested out a few detection tools to see what kind of patterns we can find in student submissions which can help us in clustering them. 

Before we show the analysis, let's discuss the type of clones and the techniques of clone detection. 

### Types of Clones:
* Type 1 or Exact clones:
 One fragment is exact copy of another fragment except of white spaces and comments.
* Type 2 or Renamed clones:
 one fragment is exact copy of another fragment except of identifier names, spaces and comments.
* Type 3 or Modified clones:
 One fragment is copied to another fragment except that some statements are added or deleted.
* Type 4 or semantic clones:
 One fragment is not syntactical similar to another fragment but behavior of two fragments are similar.


### Techniques of Code Clones Detection:

| Text Based | Token Based | Tree based | PDG based |
| --- | --- | --- | --- |
| Program is considered as a sequence of statements. Two fragments are declared as clones if they have same sequence for particular statements.  | Programs are transformed into sequence of tokens and clones are declared on the basis of matching subsequence of tokens. | Programs are converted into syntax tree or parse tree with the help of parser of the specific language. Clones are detected using different techniques applying on tree. | Program dependency graph is built which represent data flow & control flow of the program. Isomorphic matching techniques are applied for clone detection.|
| Scalability depends upon the method applied for these techniques. | Scalability is high. | Scalability depends upon algorithm applied. | Scalability for graph matching techniques are costly.|
| Easy to implement. | Less complex to implement. | Very complex to implement. | Matching techniques can be complex. |
| Technique independent of programming language. | Technique independent of programming language | Parser which is used to create ASTs is language dependent. | Independent of programming language.|
 
 We can learn more about clone detection tools in python from this link: <https://www.slideshare.net/valeriomaggio/clone-detection-in-python> 
 

We tested the tool called Clone Digger which is a tree based clone detection tool. Please find below the analysis:

![2](https://github.com/anantmittal/markdown-reports/blob/master/images/7912.png)
![3](https://github.com/anantmittal/markdown-reports/blob/master/images/7913.png)
![4](https://github.com/anantmittal/markdown-reports/blob/master/images/7914.png)  

In this analysis, we have compared two files "js_antlr.py" and "java_antlr.py". The tool was able to detect three clones amongst both the files with the diff changes in red. 

There are two dimensions of the mesh in Clone Digger: the minimal size of clones (set by --length-threshold parameter) and the maximum distance between sequences, which are reported as clones (set by --distance-threshold). There are a bunch of other parameters that we can change to find the similarity between two python codes. 

## Research Questions
Until now, if we wanted to have the snapshots of how a student has moved from attempting a problem on jupyter notebook to reaching the final solution, it was not possible. The logging extension enables the researchers to pull out such granular level data easily. These are the few research questions that can be raised now:

* Based on a typical student's sequence of k code submission attempts over time (hereby, their "trajectory") T = [AST1,AST2, ...,ASTk] on a programming exercise, predict for new students the next sequence of their code submissions within the same course.

* It is not possible to generate ASTs for student programs with syntax errors and therefore the previous feedback techniques are not applicable in repairing syntax errors. Can we use deep learning sequential models (using RNNs) to model syntactically valid token sequences using correct code submissions. Additionally, if we can query the learnt RNN model with the prefix token sequence to predict token sequences that can fix the error by either replacing or inserting predicted token sequence at the error location.
* Submissions can be clustered using density based clustering (superior to K-means). Normalized tree edit distance could be used as a distance metric. Upon receiving a submission from a student, we can grade the submission for correctness by running a suite of test cases. If the code is functionally correct, its abstract syntax tree (AST) is extracted and compared to all other submissions to determine which cluster it would best fit into. Submissions mapping to “weak” clusters (poor ABC scores) are shown approach hints, skeletons, or both.
*  Scaffolding is particularly difficult in MOOCs - hard to personalize instruction to thousands of students at once. To sort out students who might require extra attention from the instructor, we can perform the following two tasks:
    * Based on a student’s sequence of k code submission attempts over time (hereby, their ”trajectory”) T = [AST1 , AST2, ..., ASTk ] on a programming exercise, predict at the end of the sequence whether the student will successfully complete or fail the next programming exercise within the same course
    * At each t ≤ k, given a student’s sequence thus far of code submission attempts T = [AST1, AST2 , ..., ASTt ] on a programming exercise, predict whether the student will successfully complete or fail the current programming exercise
* Sequential snapshots of a student’s progress can provide valuable insights into their learning behaviour. Understanding how a student arrives at a solution is crucial to improve the effectiveness of intelligent tutoring systems.
* Do students feel more engaged when they are provided real-time feedback on their weekly assignments?

## Conclusion

Using AWS Data Pipeline, a daily scheduler exports data from DynamoDB into S3 instance. The TTL attribute in DynamoDB has been enabled which removes the items in the database based on the "expire" attribute.  

If a worthy result comes out of the research questions we have discussed, we can further extend this pipeline to process the data in real-time. This will help us in presenting meaningful feedback to the weak students and maybe motivate them to not dropout from the course midway. 

This extension is being planned to be used for few more research projects being started by Professor Brooks. 

## Questions?

Please reach out at anmittal@umich.edu or brooksch@umich.edu if you want access to the code. 