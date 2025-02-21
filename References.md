# References

[1] Zeitz, P. (1999). The art and craft of problem solving. New York: John Wiley.

[2] C. Mohan, D.L. Haderle, B. Lindsay, H. Pirahesh, and P. Schwarz. ARIES: A transaction recovery method supporting fine-granularity locking and partial rollbacks using write-ahead logging. ACM TODS, 17 (1): 94--162, 1992.

[3] Johnson, R., et al. Scalability of write-ahead logging on multicore and multisocket hardware. VLDBJ, pp. 239--263, 2012.

[4] Collin McCurdy and Jeffrey Vetter. 2010. Memphis: Finding and fixing NUMA-related performance problems on multi-core platforms. In IEEE International Symposium on Performance Analysis of Systems 8 Software (ISPASS'10). 87--96.

[5] T. Kiefer, B. Schlegel, and W. Lehner. Experimental evaluation of NUMA effects on database management systems. In BTW, 2013.

[6] A. Alquraan, H. Takruri, M. Alfatafta, and S. Al-Kiswany. An analysis of network-partitioning failures in cloud systems. In Proceedings of the 13th USENIX Conference on Operating Systems Design and Implementation, OSDI'18, pages 51--68, Carlsbad, CA, USA, 2018. USENIX Association.

[7] Adnan Alhomssi and Viktor Leis. 2023. Scalable and Robust Snapshot Isolation for High-Performance Storage Engines. Proc. VLDB Endow. 16, 6 (2023), 1426–1438.

[8] Y. Wang, M. Yu, Y. Hui, F. Zhou, Y. Huang, R. Zhu, et al.. 2022. A study of database performance sensitivity to experiment settings, Proceedings of the VLDB Endowment, vol. 15, no. 7.

[9] M. Raasveldt, P. Holanda, T. Gubner, and H. Muhleisen. Fair Benchmarking Considered Difficult: Common Pitfalls In Database Performance Testing. In 7th International Workshop on Testing Database Systems, DBTest, 2:1--2:6, 2018.

[10] M. Harchol-Balter. Performance Modeling and Design of Computer Systems: Queueing Theory in Action. Cambridge University Press, New York, NY, USA, 1st edition, 2013.

[11] Hao Xu, Qingsen Wang, Shuang Song, Lizy Kurian John, and Xu Liu. 2019. Can we trust profiling results? Understanding and fixing the inaccuracy in modern profilers. In Proceedings of the ACM International Conference on Supercomputing. 284–295.

[12] T. Mytkowicz, A. Diwan, M. Hauswirth, and P. F. Sweeney. Evaluating the accuracy of Java profilers. In PLDI, pages 187--197. ACM, 2010.

[13] https://dev.mysql.com/doc/refman/8.0/en/.

[14] Alibaba Cloud. 2019. OceanBase Did Better than Any Other Database in the TPC-C Benchmark. alibaba-cloud.medium.com.

[15] B. H. Dowden, Logical reasoning (California State University, Sacramento, CA, 2019).

[16] C. Hong, D. Zhou, M. Yang, C. Kuo, L. Zhang, and L. Zhou. KuaFu: Closing the parallelism gap in database replication. In Proc. ICDE 2013, pages 1186--1195, 2013.

[17] I. Psaroudakis, T. Scheuer, N. May, and A. Ailamaki. Task scheduling for highly concurrent analytical and transactional main-memory workloads. In ADMS Workshop, 2013.

[18] Dai Qin, Angela Demke Brown, and Ashvin Goel. 2017. Scalable replay-based replication for fast databases. Proceedings of the VLDB Endowment (2017).

[19] H. Jung, H. Han, A. D. Fekete, G. Heiser, and H. Y. Yeom. A scalable lock manager for multicores. In SIGMOD, 2013.

[20] Simon, S.: Brewer's CAP Theorem. CS341 Distributed Information Systems, University of Basel (HS2012).

[21] Harizopoulos, S. and Ailamaki, A. 2003. A case for staged database systems. In Proceedings of the Conference on Innovative Data Systems Research (CIDR). Asilomar, CA. Harizopoulos, S. and Ailamaki, A. 2003. A case for staged database systems. In Proceedings of the Conference on Innovative Data Systems Research (CIDR). Asilomar, CA.

[22] Xiangyao Yu. An evaluation of concurrency control with one thousand cores. PhD thesis, Massachusetts Institute of Technology, 2015.

[23] K. Ren, J. M. Faleiro, and D. J. Abadi. Design principles for scaling multi-core OLTP under high contention. In Proceedings of the 2016 ACM SIGMOD International Conference on Management of Data, 2016.

[24] B. Tian, J. Huang, B. Mozafari, and G. Schoenebeck. Contention-aware lock scheduling for transactional databases. PVLDB, 11(5), 2018.

[25] Peter Bailis, Alan Fekete, Michael J. Franklin, Ali Ghodsi, Joseph M. Hellerstein, and Ion Stoica. 2014. Coordination avoidance in database systems. Proc. VLDB Endow. 3 (Nov. 2014), 185--196.

[26] J. Rao, E. J. Shekita, and S. Tata. Using paxos to build a scalable, consistent, and highly available datastore. VLDB, 2011.

[27] [Paweł Olchawa](https://dev.mysql.com/blog-archive/?author=Pawe%C5%82%20Olchawa). 2018. MySQL 8.0: New Lock free, scalable WAL design. dev.mysql.com/blog-archive.

[28] Martin Kleppmann. Designing Data-Intensive Applications. English. 1 edition. O'Reilly Media, Jan. 2017. ISBN: 978-1-4493-7332-0.

[29] Heidi Howard. Distributed consensus revised. PhD thesis, University of Cambridge, 2019. URL: https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-935.pdf.

[30] Chandra, T.D., Griesemer, R., Redstone, J.: Paxos Made Live: An Engineering Perspective. In: Proceedings of the Twenty-sixth Annual ACM Symposium on Principles of Distributed Computing, New York, USA: ACM, pp. 398–407, (2007).

[31] https://dev.mysql.com/blog-archive/the-new-mysql-thread-pool/.

[32] Y. Mao, F. P. Junqueira, and K. Marzullo. Mencius: building efficient replicated state machines for WANs. In Proc. 8th USENIX OSDI, pages 369--384, San Diego, CA, Dec. 2008.

[33] C. Millsap. Thinking Clearly About Performance, Queue, vol. 8, no. 9, pp. 10-20, 2010.

[34] Leonidas Galanis, Supiti Buranawatanachoke, Romain Colle, [Benoît Dageville](https://dblp.org/pid/59/847.html), Karl Dias, Jonathan Klein, Stratos Papadomanolakis, Leng Leng Tan, Venkateshwaran Venkataramani, Yujun Wang, and Graham Wood. 2008. Oracle Database Replay. In Proceedings of the 2008 ACM SIGMOD International Conference on Management of Data. 1159--1170.

[35] G. Moerkotte and T. Neumann. Dynamic programming strikes back. In Proceedings of the 2008 ACM SIGMOD international conference on Management of data, pages 539--552. ACM, 2008.

[36] Suntorn Sae-eung. 2010. Analysis of false cache line sharing effects on multicore cpus, Master's Projects, vol. 01.

[37] D. Terry. Replicated data consistency explained through baseball, Microsoft Technical Report MSR-TR-2011-137, October 2011. To appear in Communications of the ACM, December 2013.

[38] Anirban Rahut, Abhinav Sharma, Yichen Shen, Ahsanul Haque. 2023. Building and deploying MySQL Raft at Meta. engineering.fb.com.

[39] Taipalus T. Database management system performance comparisons: A systematic survey. Published online January 3, 2023. Accessed July 31, 2023. http://arxiv.org/abs/2301.01095.

[40] Holger Pirk. 2022. https://co339.pages.doc.ic.ac.uk/decks/Profiling.pdf.

[41] M Poke. 2019. Algorithms for High-Performance State-Machine Replication. DOCTORAL DISSERTATION. HELMUT SCHMIDT UNIVERSITY.

[42] Ritwik Yadav and Anirban Rahut. 2023. FlexiRaft: Flexible Quorums with Raft. The Conference on Innovative Data Systems Research (CIDR) (2023).

[43] Jung-Sang Ahn, Woon-Hak Kang, Kun Ren, Guogen Zhang, and Sami Ben-Romdhane. 2019. Designing an efficient replicated log store with consensus protocol. In 11th USENIX Workshop on Hot Topics in Cloud Computing (HotCloud 19).

[44] Arunprasad P. Marathe, Shu Lin, Weidong Yu, Kareem El Gebaly, Per-Åke Larson, and Calvin Sun. 2022. Integrating the Orca Optimizer into MySQL. In Proceedings of the 25th International Conference on Extending Database Technology, EDBT 2022, Edinburgh, UK, March 29 - April 1, 2022. OpenProceedings.org, 2:511--2:523.

[45] https://en.wikipedia.org/wiki/.

[46] Urbán, P., Défago, X., Schiper, A.: Chasing the FLP impossibility result in a lan or how robust can a fault tolerant server be? In: 20th IEEE Symp. on Reliable Distributed Systems (SRDS), pp. 190–193. New Orleans, USA (2001).

[47] https://www.candtsolution.com/news_events-detail/what-is-the-difference-between-arm-and-x86/.

[48] N. Santos and A. Schiper. Tuning Paxos for high-throughput with batching and pipelining, in 13th International Conference on Distributed Computing and Networking (ICDCN 2012), Jan. 2012.

[49] J. Kończak, N. Santos, T. Zurkowski, P. T. Wojciechowski, and A. Schiper. JPaxos: state machine replication based on the Paxos protocol. Technical report, EPFL, 2011.

[50] R. N. Avula and C. Zou. Performance evaluation of TPC-C benchmark on various cloud providers, Proc. 11th IEEE Annu. Ubiquitous Comput. Electron. Mobile Commun. Conf. (UEMCON), pp. 226-233, Oct. 2020.

[51] Guna Prasaad, Alvin Cheung, and Dan Suciu. 2018. Improving High Contention OLTP Performance via Transaction Scheduling. https://arxiv.org/abs/1810.01997v1.

[52] Sergey Blagodurov and Alexandra Fedorova. 2011. User-level Scheduling on NUMA Multicore Systems under Linux. In in Proc. of Linux Symposium.

[53] https://www.ibm.com/docs/en/linux-on-systems?topic=management-linux-scheduling.

[54] Yanhua Mao. 2010. State Machine Replication for Wide Area Networks. Doctor of Philosophy in Computer Science. UNIVERSITY OF CALIFORNIA, SAN DIEGO.

[55] Zhou, X., Chai, C., Li, G., Sun, J.: Database meets artificial intelligence: A survey. IEEE Transactions on Knowledge and Data Engineering (2020).

[56] Prince Samuel. 2023. The Role of Logical Reasoning in Programming. medium.com.

[57] Sunny Bains. 2017. Contention-Aware Transaction Scheduling Arriving in InnoDB to Boost Performance. https://dev.mysql.com/blog-archive/.

[58] Andres Freund. 2020. Improving Postgres Connection Scalability: Snapshots. techcommunity.microsoft.com.

[59] J. M. Hellerstein, M. Stonebraker, and J. R. Hamilton. Architecture of a database system. Foundations and Trends in Databases. 1(2) pp. 141--259, 2007.

[60] https://github.com/session-replay-tools/tcpcopy.

[61] <https://github.com/session-replay-tools/cetus>.

[62] Guoliang Jin, Linhai Song, Xiaoming Shi, Joel Scherpelz, and Shan Lu. 2012. Understanding and detecting real-world performance bugs. In ACM SIGPLAN Conference on Programming Language Design and Implementation, PLDI '12, Beijing, China - June 11 - 16, 2012. 77–88.

[63] Jelena Antic, Georgios Chatzopoulos, Rachid Guerraoui, and Vasileios Trigonakis. 2016. Locking made easy. In Proceedings of the International Middleware Conference (Middleware). 1--14.

[64] https://github.com/enhancedformysql/enhancedformysql.

[65] https://github.com/enhancedformysql/benchmarksql.

[Next](Appendix.md)
