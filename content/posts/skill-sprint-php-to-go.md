---
title: "Skill Sprint PHP Monolith to Go Microservices"
draft: false
date: "2021-10-22"
---

Skill Sprint is a five-day intense learning programme led by an experienced IT professional,
where a few participating employees re-engineer specific structures or learn new technologies and practices to re-engineer
how an application works in specific situations.

## Month Before the Skill Sprint
I would say it was the most important, We have talked about three preparation-like steps.
So when we arrive at the site we will only focus on coding the solution following the best practices and standards for coding in Golang and the architecture.

### 1. Architecture
To answer this question there are two additional questions.
What are the business requirements and
Is there any tech that they follow with their monolith application.
They chose the hexagonal architecture.

The hexagonal architecture divides a system into several loosely-coupled interchangeable components, such as the application core, 
the database, the user interface, test scripts and interfaces with other systems. 
This approach is an alternative to the traditional layered architecture.


### 2. Design Patterns
After deciding on architecture we needed to decide with
which design pattern we will use that will go along best with Golang.
This is the part that will make us struggle and make us do a lot of cut-offs to make this works,
since Go's design patterns it's not very compatible with hexagonal architecture.

### 3. Project structure
Making the project structure was also a very important part so when we arrive at site
We can jump straight on coding and designing the solution for the application.

I wanted to put the architecture structure aside at the start,
so they can easily understand the Golang design pattern,
and put their main focus on best Golang practices for some time being.

## The Skill Sprint
The last week was fun, intense and one of the best experiences for me as a software engineer.

Now let's go back at the "problems".
Hexagonal architecture can have Performance overhead: adding extra components trigger extra calls to functions, therefore, in each of them we will be adding a very small overhead.
This could be a disadvantage if our service has to be extremely performant which it was the case here.

The archy was overkill for me because the micro-service has only one specific task at this point
and our main focus was on learning and coding in the best Golang practices.
The structure of the project following the hexagonal archy it's not compatible with golang concurrency patterns and structures which made us
struggle with implementing async calls in the project.
The golang interfaces are clean enough and with them, you can achieve "Hexagonal Archy" in a very much clean way and not have an impact on
speed performance.
So I would make "cutbacks" in the archy(design the archy in a different way not having the classical port, adapters, and core) to gain fast and clean code.

The advantages of the architecture for example `Separation of concerns` I would say that
they are easily managed with the well-defined Golang interfaces across the project structure,
or `Parallelization of work` , `Tests in isolation` I just simply all the time was restructuring that
with an easy well-defined Golang design pattern that was not overkilling by following this archy.

The idea of starting with a different project structure and not following the
architecture structure For Me, it was one of the best decisions they got up to speed with Golang's best practices and Go effective in three days,
and then we started with refactoring the code into the hexagonal structure.


The team adapted the Golang syntax very fast, they were really good at
catching the Golang code standards, best practices and at the same time implementing that in hexagonal archy.


This intense 5-day sprint was enough to code the two applications(micro-services) and touch everything from
simple Golang stuffs to the most complex ones -> constructing, designing and
coding concurrent tasks in the applications using Goroutines, channels, and mutexes.

An intense course of 40hours boosted and improved my soft skills, 
It was a great pleasure, and I was very happy to help and share my knowledge.

The Skill Sprint was sponsored by [Darwinist](https://darwinist.io/case-study/globalsoft/)
