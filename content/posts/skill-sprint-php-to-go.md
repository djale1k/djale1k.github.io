---
title: "Skill Sprint PHP Monolith to Go Microservices"
draft: false
date: "2021-10-22"
---

Skill Sprint is a five-day intense learning programme led by an experienced IT professional,
where a few participating employees re-engineer specific structures or learn new technologies and practices to re-engineer
how an application works in specific situations.

## 3 Weeks Before the Skill Sprint
These few weeks I would say it was the most important, We have talked about three preparation-like steps.
So when we arrive at the site we will only focus on coding the solution following the best practices and standards for coding in Golang and the architecture.

### 1. Architecture
To answer this question there are two additional questions.
What are the business requirements and
Is there any tech that they follow with their monolith application.

### 2. Design Patterns
After deciding on architecture we needed to decide with
which design pattern we will use that will go along best with Golang.
This is the part that will make us struggle and make us do a lot of cut-offs to make this works,
since Go's design patterns it's not very compatible with hexagonal architecture.

### 3. Project structure
Making the project structure was also a very important part so when we arrive on site
We can jump straight on coding and designing the solution for the application.


## The Skill Sprint
The last week was fun, intense and one of the best experiences for me as a software engineer.

Now let's go back at the problems.
Hexagonal architecture can have Performance overhead: adding extra components trigger extra calls to functions, therefore, in each of them we will be adding a very small overhead.
This could be a disadvantage if our service has to be extremely performant which it was the case here.

The archy was overkill for me because the micro-service has only one specific task at this point
and our main focus was on learning and coding in the best Golang practices.

The advantages of the architecture for example `Separation of concerns` I would say that
they are easily managed with the well-defined Golang interfaces across the project structure,
or `Parallelization of work` , `Tests in isolation` I just simply all of the time was restructuring that
with an easy well-defined Golang design pattern that was not overkilling by following this archy.

