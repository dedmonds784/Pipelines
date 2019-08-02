
# Data Pipeline

The goal of this notebook is to create a pipline framework that is flexible to different scenarios. To do this we will write classes that are not statically typed and let the user define the input and output, but lets them control the flow of the operations. 

## Inner Functions
the 'logger' function allows us to wrap a function inside another function and when assigned to a variable will create another function object that will take the arguments. This is called an inner function since the original function takes in another function as an argument and creates a new variable that will run in the 

say we have the 'logger' function that takes another functions as an argument.
Where the 'logger()' function keeps track of the functions that we run. Our logger functions looks like...


```python
def logger(func):
    def inner(*args):
        print('Calling function: {}'.format(func.__name__))
        print('With args: {}'.format(args))
        return func(*args)
    return inner
```

the problem with doing this functionally is that we would have to create a new variable each time we want to log. Say we want to add two values together and log it, our syntax will look like such...


```python
## first we define our function to run an operation on the data
def add(a, b):
    return a + b
## we then create a new variable that contains the added variable along with the logger function. 
logged_add = logger(add)
## That variable can now be used to consume the parameters of the variable to be logged
print(logged_add(1, 2))
```

    Calling function: add
    With args: (1, 2)
    3


Note that nothing was changed from the original logger function. To make this easier we can add some additional Python syntax which is the decorator symbol, '@', which will act as our wrapper. In our case it will take in the add function and log it. The new Python syntax '@' logger, created a decorator object that will modify the behavior of any function, method, or class. This use of syntax goes as follows... 


```python
## The decorator syntax '@' pro
@logger
def add(a, b):
    return a + b

print(add(1, 2))
```

    Calling function: add
    With args: (1, 2)
    3


When we wrap the 'add' function with in the '@logger' operator object, we are telling our interpretor to call the 'logger' function for each invocation of the 'add' functions. And as you can see in the example above, this gave us the same results as if were were to create a new variable that took the add function to be an argument in the logger function.

--------------------

## Class
Remember, the goal of our pipeline is to create a class that can dynamically add tasks. Writing functions works, but forces us to retype our pipeline each time. Instead of using functions as decorators though, we can use our instance methods. The below script is an example of how this behavior will look...
```
pipeline = Pipeline()

@pipeline.task()
def first_task(x):
    return x + 1

print(pipeline.tasks)
```

To enable this behavior the first thing we need to create is our 'Pipeline' class. This class will hold our methods we will use with decorators. Our class will be defined as such...


```python
class Pipeline:
    def __init__(self):
        self.tasks = []
        
    def task(self):
        def inner(f): 
            self.tasks.append(f)
        return inner 
```

the initialization, '__init__' method are the first tasks that run when a variable is created using the class. In this case our creating a variable of the object using the 'Pipeline' class will initialize a new variable to have an empty 'task' list. Meanwhile, the 'task' method is one of the instance methods that will feed to a decorator and then will be append that function call to a list each time it is called. Example...


```python
## create pipeline variable using the 'Pipeline' class
pipeline = Pipeline()

## wrap the function 'first_task' in our method using the decorator
@pipeline.task()
def first_task(x):
    return x + 1

## print the 'tasks' object stored in the pipeline variable to see all values in the list.
print(pipeline.tasks)
```

    [<function first_task at 0x1042ca510>]


Again, this gave us the same results as before, but much more dynamically. Now each time we call the 'first_task' function an instance of its generator  object will be dynamically added to a list.

## Work Flow
Ultimatley, we want our data pipeline to be aware of all the tasks that have been ran. Since we have the ability to call a decorator before wrapping, this means it is possible to pass in optional arguments before wrapping as well. The behavior  we are aiming towards looks somewhat like this... 

```
pipeline = Pipeline()

@pipeline.task()
def first_task(x):
    return x + 1

@pipeline.task(depends_on=first_task)
def second_task(x):
    return x * 2

print(pipeline.tasks)
```

Where we wrap a function in a decorator to be dynamically added to our tasks list, and then we  wrap a second function in a decorator, but this time it keeps in memory that the first task must run before the second task.

Essentially, we want the tasks to be exectuted in an ordered format. The order in the pipeline example above makes the running of the 'second_task' conditional on the fact that the 'first_task' has ran. The 'depends_on' keyword enforces this ordering, so that we can determine the dependency link of each task.

To enable this functionality, we need to rewite the class to have a the 'depends_on' argument added to the task method.


```python
class Pipeline:
    def __init__(self):
        self.tasks = []
    ## 'task' method runs only depending on if the condition is met if one is set
    def task(self, depends_on=None):
        idx = 0
        if depends_on:
            idx = self.tasks.index(depends_on) + 1
        def inner(f):
            self.tasks.insert(idx, f)
            return f
        return inner
```

The new 'Pipeline' class above now contains an argument 'depends_on' that will check to see if a task  is in the pipleline before running it. If the task is in the 'tasks' list it will locate its position in the list using the 'index' method and add it the the position right after that task. 

If a task being added to the list is not contingent on another task running, it will skip editing the 'idx' value and simply insert the task to the 'tasks' list. This addition to the method allows the user to add an element of organization to the pipeline.


```python
# create a pipeline variable by calling the 'Pipeline' class
pipeline = Pipeline()

# append 'first_task to the list'
@pipeline.task()
def first_task(x):
    return x + 1

## append 'second_task' to the list with 'first_task' assigned to the 'depends_on' argument
@pipeline.task(depends_on=first_task)
def second_task(x):
    return x * 2

## Append 'third_task' to the list with 'second_task' assigned to the 'depends_on' argument
@pipeline.task(depends_on=second_task)
def last_task(x):
    return x - 4

print(pipeline.tasks)
```

    [<function first_task at 0x1042cad08>, <function second_task at 0x1042caae8>, <function last_task at 0x1042ca268>]


As seen in the output from the 'tasks' object from our 'pipeline' variable, we can see that the decorators add each of the functions to our list based on their order given by the 'depends_on' argument.

With a framework in place for how functions will be added to the pipeline in an orderly manner, we can now add a new method to our class in order to execute the functions in the pipeline. The method we are going to run will be called 'run' where the goal will to place the output from the previous function into the following functions.


```python
class Pipeline:
    def __init__(self):
        self.tasks = []
        
    def task(self, depends_on=None):
        idx = 0
        if depends_on:
            idx = self.tasks.index(depends_on) + 1
        def inner(f):
            self.tasks.insert(idx, f)
            return f
        return inner
    ## Our new run method iterativley runs each method and places the output from the 
    # previous function into the function that is up next
    def run(self, input_):
        output = input_
        for task in self.tasks:
            output = task(output)
        return output
```

Our class now contains a 'run' method with an output variable. The output starts the loop as the value the user inserts to the 'input_' parameter. the output variable is then assigned to the be an argument for the first task in the pipeline. When the first task is ran the function then assigns the output to be the new value assigned to our 'output' variable to be inserted into as the input for the task that is upnext in the iteration. This then runs untill completition. 

Our execution behavior will then look like... 


```python
pipeline = Pipeline()
    
@pipeline.task()
def first_task(x):
    return x + 1

@pipeline.task(depends_on=first_task)
def second_task(x):
    return x * 2

@pipeline.task(depends_on=second_task)
def last_task(x):
    return x - 4

print(pipeline.run(20))
```

    38


The addition of the 'run' method enables the completition of our fully functioning general pipeline. This  pipeline can be used with any set of tasks that require a dependency ordering.
