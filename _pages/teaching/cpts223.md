---
layout: page
title: CPTS 223 – Advanced Data Structure C/C++
permalink: /teaching/cpts223/
---

## Course Overview

CPTS 223 is a computer science course for majors. In this course, we use the C++ programming language to design, develop, implement and analyze data structures and algorithms beyond the basic data structures discussed in CptS 122. This course also introduces you to algorithm de- sign through different data structures and algorithmic techniques, and algorithm analysis using asymptotic notation and complexity analysis. One important component of this course is to have you design multiple algorithmic solutions to a single problem using different combinations of data structures that you learn in class, and then comparatively evaluate their relative efficiencies. Another important component of this class is to have you implement the data structures and algorithms using C++ and STL (Standard Template Library), and evaluate the algorithms em- pirically. You will also learn basic concepts in parallel and multithreaded programming such as how to analyze parallel runtime complexity, compute speedup; and synchronization primitives such as locking. You will be introduced to OpenMP multithreaded programming for shared memory multicore systems. You will have at least one programming project that expects you to come up with a parallel implementation for one of the data structures – e.g., parallel hash table construction and access. You will also learn to include parallel performance statistics (such as speedup) in results. This course will introduce gcc/CMake on the Linux platform as the development environment. Tools such as Git will be introduced for tracking source versions and will be the primary vehicle for turning in assignments. All projects will be tested on Ubuntu Desktop
20.04/22.04 (LTS) or macOS.


## Syllabus

- 📄 **[Course Syllabus (PDF)](https://drive.google.com/file/d/1AORaiUhCHPrv3xLYhbsexvBdDAP7nhaG/view?usp=sharing)**


## Lecture Schedule

| Lecture | Topic | Slides |
|-----:|-------|--------|
| 1  | \[0\] Why Data Structure 1 | [Slides](https://drive.google.com/file/d/1rbK--UBx6ONIoOAZMyMKIyuTZ40L4-Js/view?usp=sharing) |
| 2  | \[1\] C++ Tools  | [Slides](https://drive.google.com/file/d/1TKYBVh21svNCe8kxxehsfPRuFxI9sBYT/view?usp=sharing) |
| 3  | \[1\] C++ Features  | [Slides](https://drive.google.com/file/d/1A09UsAONk0F1aJl-ERlk_WQlb8tEeY7U/view?usp=sharing) |
| 4  | \[2\] Basic ADTs  | [Slides](https://drive.google.com/file/d/1NhhJ4GrOok0iwpqq2h3L2Td14TwsY8ZR/view?usp=sharing) |
| 5  | \[3\] Why Data Structure 2 | [Slides](https://drive.google.com/file/d/1cvB0hh1_Us4Bs_jLGgLysEVTEyqKaG88/view?usp=sharing) |
| 6  | \[3\] Math Review: Basic | [Slides](https://drive.google.com/file/d/1eJzasuF5a8Xd3SndZ3vHPBkj5uyROdmG/view?usp=sharing) |
| 7  | \[3\] Math Review: Proof  | [Slides](https://drive.google.com/file/d/17NbUXt6wrgWs6OcesG72TjJ6ik1rwrdv/view?usp=sharing) |
| 8  | \[4\] Algorithm Analysis  | [Slides](https://drive.google.com/file/d/1SbeZDH69OsIxElLrbdc4im-9iKp235UN/view?usp=sharing) |
| 9  | \[5\] BST  | [Slides](https://drive.google.com/file/d/1Ga-rep9_L0NxoUWLshkB6m8nNYvzM7Fj/view?usp=sharing) |
| 10 | \[5\] AVL  | [Slides](https://drive.google.com/file/d/1U7dVcMQnuJS2Scz0XupWQhVP1SCAnsVi/view?usp=sharing) |
| 11 | \[5\] Splay Tree  | [Slides](https://drive.google.com/file/d/1-sO2aRz03yTMbAiofdWiwRddIiMSS7Fc/view?usp=sharing) |
| 12 | \[5\] B-Tree  | [Slides](https://drive.google.com/file/d/1yFPF_VSYPQj-qamNl2f5Jst7HE99c96n/view?usp=sharing) |
| 13 | \[5\] Set and Map  | [Slides](https://drive.google.com/file/d/1BvoJI-ipjBjsnjhHQXLW0cRGOy6E7H8p/view?usp=sharing) |
| 14 | \[5\] RB Tree  | [Slides](https://drive.google.com/file/d/1ascyPdCe0djKWbuZA3grKoJS24PP8HT0/view?usp=sharing) |
| 15 | \[6\] Hash Table  | [Slides](https://drive.google.com/file/d/1EK_654k1dQKhib0fWmBSX0r2rCOqDwmT/view?usp=sharing) |
| 16 | \[7\] Parallel Computing 1  | [Slides](https://drive.google.com/file/d/17dI8CmbulbQ6OcLdpZsofBLMpTWxfVFq/view?usp=sharing) |
| 17 | \[7\] Parallel Computing 2  | [Slides](https://drive.google.com/file/d/1XIxLvG9di_vkxzjz5l4lGxTZaNDBdTJt/view?usp=sharing) |
| 18 | \[7\] Parallel Computing 3  | [Slides](https://drive.google.com/file/d/1L59QsqCzTMmjXRvtqjHWvE_dDGsISw6z/view?usp=sharing) |
| 19 | \[8\] Heaps  | [Slides](https://drive.google.com/file/d/13bV0DmvjRESe91A6sf4wf3Epz7ropc7M/view?usp=sharing) |
| 20 | \[9\] Sorting  | [Slides](https://drive.google.com/file/d/16jXBgC8jbat3K0byJA2VidHPu0Zexy4u/view?usp=sharing) |
| 21 | \[10\] Disjoint Sets  | [Slides](https://drive.google.com/file/d/1QS2pxR6IT77DdJbdxBSMy7bGtwLNwKiX/view?usp=sharing) |
| 22 | \[11\] Graph  | [Slides](https://drive.google.com/file/d/1xZHnzvcSQQkVjrnFchGKqVIalJfOiog4/view?usp=sharing) |
| 23 | \[12\] Data Structure for AI  | [Slides](https://drive.google.com/file/d/13o8ymyMtYvzHIE00QGvHwMWqACX8ojjv/view?usp=sharing) |





