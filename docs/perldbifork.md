---
layout: docs
title: perldbifork
prev_section: perlreftut
next_section: perlobj 
permalink: /docs/perldbifork
---
<div class="note ">
    <h5>perl DBI fork</h5>
    <p>
    Perl fork,DBI,clone
    </p>
</div>
<div id="toc"> </div><br>

# 坑
在之前的一段时间里,因为个人需要我需要对几百万的数据进行 CRUD 操作,鉴于对 AnyEvent::DBI 的靠谱程度不是很信赖,所以
我使用了 fork 多进程来处理的方案,但是实际操作中还是不小心掉了几个坑里,现在总结,防止以后重复掉坑:

* 父子进程共享 dbi 句柄

unix 环境下 fork 进程对文件句柄是 share 的,也就是说 dbi 句柄也是共享的,假设我们父进程和子进程同时进行一个 select
的查询,那么恭喜你很可能掉坑里了,例如:

    # father process
    SELECT * FROM foobar WHERE baz = 3

    # child process
    SELECT grzbaka FROM baz WHERE foo = 5

现在,我们实际的查询可能会变成这样

    SELECT * FROSELECT grzbaka FROM baM foobar WHERE baz = 3z WHERE foo = 5

当然实际的情况还是会更复杂的,除非你使用的是 AnyEvent::DBI 或者另外的 POE 之类的事件型模块.
所以争取的代码就是在子进程创建新的 db 链接或者使用 dbh 的 clone 方法:
    
    use DBI;
    use strict;
    use warnings;

    my $pid = fork;
    if( $pid ){
        my $dbh = DBI->connect($dsn,$user,$password);
    }else{
        my $child_dbh = DBI->connect($dsn,$user,$password);
        #or 
        $child_dbh = $dbh->clone();
    }

或者你使用Parallel::ForkManager:

    # this is just example script
    use strict;
    use warnings;
    use Parallel::ForkManager;
    use DBI;

    my @links=(1..1000);
    my $pm=new Parallel::ForkManager(5);

    foreach my $x (0..$#links) {
        $pm->start and next;
            my $dbh = DBI->connect("DBI:mysql:database=customers;host=loca
    +lhost;port=3306", "root", "")
            or die "Can't connect: ", $DBI::errstr; 
          $dbh->{RaiseError} = 1;
        print "$links[$x]\n";
          # do something (read, update, insert from db)
          undef $dbh;
        $pm->finish;
    }

* 在子进程 undef 掉父进程的 $dbh

boy,这是非常危险的操作哦,这会爸父进程的 dbh 也干掉,因为他们并不是彼此的 copy,他们是同一个句柄,实际操作中,我们可以这样做的:
    
    my $child_dbh = $dbh->clone;
    # in child process
    $child_dbh->{InactiveDestroy} =1;
    undef $dbh;

InactiveDestroy 这个设置就是告诉 DBI 如果子进程的 dbi 被销毁了,父进程的 dbh 不要关闭.(前提是你这个$dbh 是克隆的句柄)


# 测试
为了验证以上诸坑,我从网上 copy 了一段 testmore 的代码

    #!/usr/bin/perl

    use strict;
    use warnings;

    use Test::More 'no_plan';
    use DBI;

    my @connect_parameters
        = ( 'DBI:Pg:dbname=template1', 'user', 'pass',
            { ShowErrorStatement => 1,
              AutoCommit => 1,
              RaiseError => 1,
              PrintError => 0, } );

    # An earlier version leaked socket connections
    # and would eventually fail (or cause "Too many
    # open files" errors elsewhere).  I loop a while
    # here to detect that bug.
    foreach my $iteration ( 1 .. 2_000 ) {

        warn "iteration $iteration\n";

        my $dbh = DBI->connect( @connect_parameters );
        isa_ok( $dbh, 'DBI::db' );

        my $one;
        ($one) = $dbh->selectrow_array( 'SELECT 1' );
        is( $one, 1, 'can select 1' );

        # this is fetched later
        my $sth = $dbh->prepare( 'SELECT 1' );
        $sth->execute;

        ok( ! $dbh->{InactiveDestroy},
            'dbh InactiveDestroy is off before fork' );

        my $pid = fork();
        if ( ! defined $pid ) {
            die "Can't fork: $!\n";
        }

        if ( $pid ) {
            # parent

            isa_ok( $dbh, 'DBI::db' );
        
            ($one) = $dbh->selectrow_array( 'SELECT 1' );
            is( $one, 1, 'parent can select 1 before child exits' );

            is( wait(), $pid, 'waited for child' );

            ($one) = $dbh->selectrow_array( 'SELECT 1' );
            is( $one, 1, 'parent can select 1 after child exits' );
        }
        else {
            # child

            my $child_dbh = $dbh->clone();
            isa_ok( $dbh, 'DBI::db' );
            isa_ok( $child_dbh, 'DBI::db' );

            ok( ! $dbh->{InactiveDestroy},
                'dbh InactiveDestroy is off in child after fork' );
            ok( ! $child_dbh->{InactiveDestroy},
                'child_dbh InactiveDestroy is off in child after fork' );

            $dbh->{InactiveDestroy} = 1;

            ok( $dbh->{InactiveDestroy},
                'dbh InactiveDestroy is on in child after fork' );
            ok( ! $child_dbh->{InactiveDestroy},
                'child_dbh InactiveDestroy is off in child after fork' );

            undef $dbh;
            ok( ! $dbh, 'death to dbh in child' );

            ($one) = $child_dbh->selectrow_array( 'SELECT 1' );
            is( $one, 1, 'child can select 1' );

            exit;
        }

        ($one) = $sth->fetchrow_array;
        is( $one, 1, 'select running before fork still works' );
    }

总而言之,就是处理 dbh fork 的时候要注意句柄是共享的,不是 copy 的.










