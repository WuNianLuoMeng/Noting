# redis-事务

Redis通过MULTI，EXEC，WATCH等命令来实现事务的功能。事务首先以一个MULTI命令开始，接着将多个命令放入事务当中，最后由EXEC命令将这个事务提交。

