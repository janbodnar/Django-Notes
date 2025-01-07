# Definitions

## Failures, faults, errors

A failure is a difference from the expected result. This is the problem we observe. A fault is the  
cause of the failure. (For instance, a developer's mistake or an omission in planning.)  
An error is the actual mistake which caused the fault to occur. (A typo, a stack overflow, an SQL injection.)  


## Processes 

 - *Linting*: the process of running a program that statically analyses code for potential errors.  
   The code in question is not run.  
 - *Debugging*: debugging is the process of identifying and removing errors from a computer program.  
   It localizes and corrects faults.  
 - *Software testing*:  process of checking the quality, functionality, and performance of the software  
   under test by validation and verification. The goal of testing is the systematic detection of failures.  

Linting and debugging is usually done by developers. Testing is done by testers. 


## Test types

- Unit test: checks a small bit of code, like a function or class. For instance, we check the validity
  of a price calculation.  
- Integration test: is a level of software testing where individual units/components  
  are combined and tested as a group. (We check if a function saves data into a database table.)  
- End-to-end test: a testing method that evaluates the entire application flow, from start to finish.
- Functional test: checks a single bit of software functionality, such as addition or deletion of a user.
- Regression test: conducted after a code update to ensure that the update introduced no new bugs.
- Performance test: how software performs under different workloads.
- Stress test: how much strain the system can take before it fails.

### Functional vs nonfunctional testing  

Unit Testing, integration testing, and end-to-end testing are all types of functional testing.  
Non-functional testing comprises the behaviour aspect of the system, i.e., performance, stress, etc.  
Performance testing, usability testing, and volume testing are all types of non-functional testing.  

## Beta testing 

Beta testing is the process of testing a software product or service in a real-world environment  
before its official release. During beta testing, the software is made available to a selected 
group of users who are willing to test the product and provide feedback to the developers.  


## Test case 

A test case is a document which consists of a set of conditions or actions which are performed  
on the software application in order to verify the expected functionality of the feature.  

    Test Case ID 
    Test case Description/Summary
    Test steps 
    Pre-requisites 
    Test category
    Author
    Automation 
    Pass/fail
    Remarks

## Test suite 

A test suite is a collection of test cases. In automated testing, it can mean a collection of test scripts.  
In a test suite, the test cases / scripts are organized in a logical order.  

Example of a test suite for purchasing a product, having 4 test cases:

- Login
- Add Products
- Checkout
- Logout



## Test coverage

Test coverage is a software testing metric that indicates the quantity of testing completed  
by a collection of tests. It helps identify areas that are missing or not validated.


## Smoke testing 

Smoke testing is a type of software testing that is typically performed at the beginning of the  
development process to ensure that the most critical functions of a software application are   
working correctly. It is used to quickly identify and fix any major issues with the software   
before more detailed testing is performed. The goal of smoke testing is to determine whether   
the build is stable enough to proceed with further testing. 

- performed early in the development process
- The goal is to quickly identify and fix major issues with the software
- It tests the most critical functions of the application
- Helps to determine if the build is stable enough to proceed with further testing
- Synonyms: confidence testing, build verification testing, or build acceptance testing


Smoke testing is also done by testers before accepting a build for further testing.

Smoke tests are a subset of test cases that cover the most important functionality of a component  
or system, used to aid assessment of whether main functions of the software appear to work correctly.  
Microsoft claims that after code reviews, smoke testing is the most cost-effective method for 
identifying and fixing defects in software.  

One can perform smoke tests either manually or using an automated tool.  

Smoke testing originated in the hardware testing practice of turning on a new piece of hardware  
for the first time and considering it a success if it does not catch fire and smoke. 


## User Acceptance Testing

User Acceptance Testing (UAT) is a type of testing performed by the end user or the client  
to verify/accept the software system before moving the software application to the production  
environment. UAT is done in the final phase of testing after functional, integration  
and system testing is done.


## Code review

Code review (sometimes referred to as peer review) is a software quality assurance activity in  
which one or several people check a program mainly by viewing and reading parts of its source code.  

Goals 

- Better code quality  – improve internal code quality and maintainability
  (readability, uniformity, understandability, etc.)
- Finding defects  – improve quality regarding external aspects, especially correctness,
  but also find performance problems, security vulnerabilities, injected malware, ...
- Learning/Knowledge transfer  – help in transferring knowledge about the codebase, solution approaches,
  expectations regarding quality, etc.; both to the reviewers as well as to the author
- Increase sense of mutual responsibility and solidarity
- Finding better solutions  – generate ideas for new and better solutions 
- Complying to QA guidelines, ISO/IEC standards  – Code reviews are mandatory in some contexts, e.g.,
  air traffic software, safety-critical software
