# debian-from-scratch

An instruction manual for teaching Linux From Scratch users how to make a custom Debian-powered distro.

##Why Debian from Scratch?

The original Linux from Scratch manual is purposefully vague as to what technique one should use to manage software dependencies. The suggestions that it gives, while no doubt being interesting exercises in package management, are not necessarily heavy-duty answers to a system administrator who intends on managing his time efficiently. 

The disadvantage of compiling everything to create a fully-fledged system is time. After one builds an LFS system for the first time, he/she is apt to realize that managing dependencies can be an arduous task, to say the least. Going through the insufferable exercise of hunting down dozens to possibly hundreds packages, mapping dependencies, configuring and installing these dependencies in the correct order, just to install a single piece of software, is not a viable alternative to a system administrator who values his time.

The answer to this problem, is obviously to use a package manager. There are many available package managers, the most popular of which are the Debian-based package management suites (dpkg and apt), and the Red Hat based package management suites (rpm and yum). 

This manual will teach you how to build your system utilizing the Debian set of package management tools by utilizing the temporary system environment created in Linux From Scratch.

## Purpose for this project

I chose to make this manual because I have seen woefully old guides on the internet teaching others how to get dpkg and apt running on their own custom linux, and people asking on various forums on how to install dpkg and apt, but not getting the help that they need. These guides are outdated and no longer contain up-to-date information, which I intend to fix here in this manual.

This project intends to be a community resource to help those interested in creating their own custom system from the ground up, while fully taking advantage of the Debian suite of package management, dpkg and apt, in order to solve the problems of package dependency installation and management.

## How to use this manual? 

This manual is designed to be used after completing all instructions up to the end of Chapter 5 of the Linux From Scratch book, version 7.9. One first follows the instructions of the original LFS book, and builds the temporary system that is created in LFS Chapter 5. One is required to have a fully-functional temporary system which the the outcome of Chapter 5. 

After completing the preliminary preparartions, one then consults this manual and follows it step-by-step.

Like the original LFS manual, when dealing with packages to be compiled, each section already assumes that you have extracted the source code and have changed your main directory into the main folder of the extracted content. However, when dealing with .deb files no such extraction is needed. One only needs to follow the instructions while having the .deb file in your current directory.

