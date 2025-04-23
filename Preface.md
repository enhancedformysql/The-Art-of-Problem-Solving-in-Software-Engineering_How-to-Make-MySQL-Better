# Preface

## Who This Book Is For?

When software professionals encounter complex problems, they often face contradictions and struggle to find solutions. To overcome these challenges, it's crucial to view tough problems as opportunities for significant rewards. Embracing difficult challenges can lead to valuable insights and breakthroughs.

With 20 years of problem-solving experience, most challenging problems in server-side software can be solved through systematic logical analysis. Despite its global prominence, MySQL faces a variety of problems. The journey in addressing these MySQL problems has enriched understanding and led to the development of unique insights, which inspired the writing of this book.

This book uses MySQL challenges as case studies to explore problem analysis and resolution strategies. Readers will gain a deeper appreciation for logical reasoning, data structures, algorithms, and more through practical examples and insightful discussions.

This book covers the following topics:

1. **Logical Reasoning for MySQL**: Techniques for analyzing complex MySQL problems using logical reasoning.
2. **Computer Science Fundamentals**: Fundamentals of computer science relevant to MySQL.
3. **MySQL Internals**: An introduction to the basics of MySQL core components.
4. **Performance Testing**: Methods for scientifically testing MySQL performance.
5. **MySQL 8.0 Improvements**: Key improvements in MySQL 8.0 compared to MySQL 5.7.
6. **MySQL Improvements**: Further optimizations in a standalone MySQL instance.
7. **Group Replication Advancements**: Notable enhancements in MySQL Group Replication.
8. **Secondary Replay improvements**: Optimizations in MySQL's secondary replay.
9. **Performance Optimization Techniques**: Strategies for optimizing MySQL applications.

**Focus of the Book:**

Building on the goal of enhancing MySQL scalability, this book explores various optimization strategies to increase throughput, reduce response times, and ensure that MySQL secondaries closely match the performance of the primary server, thereby achieving high availability failover with very little delay in most scenarios.

**Target Audience:**

1. **Quality-Focused Developers**: Individuals committed to high-quality software development.
2. **Problem Solvers**: Those interested in tackling complex software problems.
3. **MySQL Enthusiasts**: Readers looking to understand MySQL from the ground up.
4. **Testing Practitioners**: Users keen on mastering effective testing techniques.
5. **Software Architects**: Those designing large-scale software systems.
6. **Performance Engineers**: Engineers struggling with performance optimization.
7. **Computer Science Students**: Learners aiming to strengthen their foundational knowledge.
8. **MySQL Researchers**: Researchers conducting studies based on MySQL.

## Outline of This Book

Part 1 focuses entirely on MySQL-related problems. Chapter 1 explores how users approach and solve challenging MySQL problems. Chapter 2 focuses on mysterious problems in MySQL 8.0.

Part 2 primarily covers the fundamental knowledge related to problem-solving. Chapter 3 emphasizes the importance of logical reasoning skills in addressing MySQL challenges effectively. Chapter 4 provides practical computer fundamentals essential for understanding MySQL problems. Chapter 5 introduces MySQL kernel basics, laying the groundwork for subsequent problem-solving discussions. Chapter 6 covers scientific methods for testing MySQL.

Part 3 primarily focuses on the analysis and improvement of specific problems. Chapter 7 highlights notable optimizations in MySQL 8.0. Chapter 8 examines improvements for solving problems specific to MySQL 8.0. Chapter 9 discusses enhancements in MySQL Group Replication, while Chapter 10 focuses on improvements in MySQL secondary replay. Together, these chapters not only boost MySQL performance but also lay the foundation for achieving a high availability cluster.

Part 4 primarily analyzes performance improvements from the perspective of MySQL users. Chapter 11 presents techniques to improve MySQL 8.0 application performance without code changes.

Part 5 is the concluding summary. Chapter 12 outlines future directions for MySQL improvements and concludes the book.

## References and Further Reading

This book focuses on analyzing and solving MySQL problems, so a certain level of computer science background is recommended. To support understanding and maintain continuity, key terminology is included in the "Glossary" section of the appendix. For those lacking a foundation in MySQL, please refer to the related content in the appendix or consult dedicated MySQL books [69].

## Special Terminology Explanation

1. **Balanced Replay Speed**
   
   For MySQL secondary replay, this defines balanced replay speed where the MySQL secondary matches the primary under normal circumstances. When the speed is at or below the balanced replay speed, there is no significant lag in transaction replay progress on the secondary. However, if the speed exceeds this threshold, the secondary begins to lag behind the primary in transaction replay progress.

2. **Dual One**
   
   Specifically refers to two major configurations in MySQL: *sync_binlog=1* and *innodb_flush_log_at_trx_commit=1*.

3. **MySQL Secondary Replay**
   
   This term represents the common replay process for asynchronous replication, semisynchronous replication, and Group Replication. In this book, 'MySQL primary' is consistently used to represent the MySQL source, and 'MySQL secondary' to represent the MySQL replica.

## Deployment Supplementary Explanation

When deploying MySQL for testing, it is preferable to match the test environment as closely as possible to the production environment, unless certain configurations significantly interfere with performance analysis. Below are specific declarations related to MySQL deployment during the testing process:

1. The mainstream servers in use all have a 4-way NUMA architecture.
2. Servers are generally x86 architecture.
3. NVMe SSD hardware disks are used, and the I/O latency is very low.
4. All tests were conducted on high-performance servers and only represent results for high-performance server configurations.
5. All tests are conducted on the Linux operating system.
6. MySQL standalone tests typically use version 8.0.27, while MySQL cluster tests generally use version 8.0.32.
7. Improvements to MySQL are referred to as improved MySQL or modified MySQL.
8. The transaction isolation level in TPC-C tests is *Read Committed*.
9. The storage engine used for transactions is InnoDB.
10. Unless otherwise stated, the default mode during MySQL Group Replication testing is single-primary mode.
11. The *binlog_format* parameter is set to row-based format.
12. MySQL, whether primary or secondary, uses GTID (Global Transaction Identifier).
13. Cluster settings include *replica_preserve_commit_order=on*.
14. Most tests are based on BenchmarkSQL TPC-C.
15. Unless stated otherwise, TPC-C tests are generally based on partitioned large tables.
16. Due to the extensive number of tests, the related software, hardware, and configurations are detailed in the Appendix.

## How to Contact Us

For configurations, patches, and additional information related to this book, please visit our webpage: <https://github.com/enhancedformysql/>.

Email [wangbin579@gmail.com](mailto:wangbin579@gmail.com) to comment or ask technical questions about this book.

## Acknowledgments

This book meticulously organizes a wealth of ideas and knowledge contributed by numerous individuals, encompassing insights from both academic research and industrial practice. In computing, there is a natural inclination towards the latest innovations, yet our historical foundations offer valuable lessons. This book references dozens of articles, blog posts, documentation, and other resources, making it an invaluable learning resource for me. I am profoundly grateful to the authors for their generous contributions of knowledge.

Several individuals have been crucial in the writing of this book by reviewing drafts and offering feedback. I am especially grateful for the contributions of Hongshen Wang, Jinrong Ye, Riyao Gao, and Haitao Gao. Naturally, I take full responsibility for any remaining errors or contentious opinions in this book.

Finally, my deepest gratitude to my family, whose unwavering support has been indispensable throughout this nearly three-month writing journey. You are the best.

[Next](Part1.md)
