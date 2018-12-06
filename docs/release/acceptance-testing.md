Acceptance Testing
==================

The OSG Release Team collects and maintains testing procedures for major components in the OSG Sofware Stack. These test should verify that basic functionality of the component works in typically deployed configurations.

CVMFS
-----

!!! note
    This acceptance testing recipe was created when access to a machine with sufficient disk space to make a complete replica of OASIS was not available.

### Creating a CVMFS Repository Server (Stratum 0) ###

1.  Disable SELinux by setting the following in `/etc/selinux/config`.

        SELINUX=disabled
    
2. Check kernel version.

        :::console
        uname -a 

3. CVMFS for EL7 requires OverlayFS (as of kernel version 4.2.x). If default kernel is <= 4.2.x, update kernel.

        :::console
        root@host # rpm --import <https://www.elrepo.org/RPM-GPG-KEY-elrepo.org>
        root@host # rpm -Uvh <http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm>
        root@host # yum install yum-plugin-fastestmirror
        root@host # yum --enablerepo=elrepo-kernel install kernel-ml 

4. Select updated kernel by editing `/etc/default/grub`.

        GRUB_DEFAULT=0
        
    and run:

        :::console
        root@host # grub2-mkconfig -o /boot/grub2/grub.cfg 

5. Reboot system. 
6. Check kernel version again and make sure SELinux is disabled.

        :::console
        root@host # uname -a
        root@host # sestatus

7. If kernel >= 4.2 and SELinux is disabled, then update system and install CVMFS server and client packages.

        :::console
        root@host # yum update
        root@host # yum install epel-release
        root@host # yum install yum-plugin-priorities
        root@host # rpm -Uvh <https://repo.opensciencegrid.org/osg/3.3/osg-3.3-el7-release-latest.rpm>
        root@host # yum install cvmfs cvmfs-server

8. Configure web server and start it up. Edit `/etc/httpd/conf.d/cvmfs.conf`:

        Listen 8000
        KeepAlive On

    and run:

        :::console
        root@host # chkconfig httpd on
        root@host # service httpd start 

9. Make new repository.

        :::console
        root@host # cvmfs_server mkfs test.cvmfs-stratum-0.novalocal 

10. Run transaction on new repository to enable write access.

        :::console
        root@host # cvmfs_server transaction test.cvmfs-stratum-0.novalocal

11. Place some sample code in new repository directory and then publish it.

        :::console
        root@host # cd /cvmfs/test.cvmfs-stratum-0.novalocal
        root@host # vi [bash\_pi.sh](%ATTACHURL%/bash_pi.sh)
        root@host # chmod +x bash\_pi.sh
        root@host # cvmfs\_server publish test.cvmfs-stratum-0.novalocal 

12. Check repository status after publication.

        :::console
        root@host # cvmfs\_server check
        root@host # cvmfs\_server tag
        root@host # wget -qO- <http://localhost:8000/cvmfs/test.cvmfs-stratum-0.novalocal/.cvmfswhitelist%7Ccat> -v

13. Download a copy of the CVMFS repository's public key e.g., /etc/cvmfs/keys/test.cvmfs-stratum-0.novalocal.pub

### Creating a CVMFS Replica Server (Stratum 1) ###

1.  Repeat steps 1 though 8 in the previous section on "Creating a CVMFS Repository Server ". However, now also install `mod_wsgi`.
    
        :::console
        root@host # yum install cvmfs cvmfs-server mod\_wsgi 

2. Upload a copy of the CVMFS repository's public key and place in `/etc/cvmfs/keys` directory. 
3. Add replica of the repository.
    
        :::console
        root@host # cvmfs_server add-replica -o root <http://10.128.3.96:8000/cvmfs/test.cvmfs-stratum-0.novalocal> /etc/cvmfs/keys/test.cvmfs-stratum-0.novalocal.pub 
    
4. Make a snapshot of the repository.

        :::console
        root@host # cvmfs\_server snapshot test.cvmfs-stratum-0.novalocal

### Creating a CVMFS client ###

1.  Update system and install CVMFS client package.

        :::console
        root@host # yum update
        root@host # yum install epel-release
        root@host # yum install yum-plugin-priorities
        root@host # rpm -Uvh <https://repo.opensciencegrid.org/osg/3.3/osg-3.3-el7-release-latest.rpm>
        root@host # yum install cvmfs

2. Upload a copy of the CVMFS repository's public key and place in `/etc/cvmfs/keys` directory.
3. Edit fuse configuration `/etc/fuse.conf`.


        user_allow_other

4. Edit autofs configuration and restart service `/etc/auto.master`.

        /cvmfs /etc/auto.cvmfs
        
    and run:

        :::console
        root@host # service autofs restart

5. Edit cvmfs configuration (`/etc/cvmfs/default.local`) to point to replica server.

        CVMFS_SERVER_URL="http://10.128.3.97:8000/cvmfs/@fqrn@"
        CVMFS_REPOSITORIES="test.cvmfs-stratum-0.novalocal"
        CVMFS_HTTP_PROXY=DIRECT

6. Remove OSG CVMFS configuration file.

        :::console
        rm /etc/cvmfs/default.d/60-osg.conf

7. Run CVMFS config probe.

        :::console
        cvmfs_config probe test.cvmfs-stratum-0.novalocal

8. Check CVMFS config status.

        :::console
        cvmfs_config stat -v test.cvmfs-stratum-0.novalocal

9. Execute sample code published to repository from client.
    
        :::console
        /cvmfs/test.cvmfs-stratum-0.novalocal/bash_pi.sh -b 8 -r 5 -s 10000

### Creating an OASIS client ###

1.  Update system and install CVMFS client package.
    -   yum update
    -   yum install epel-release
    -   yum install yum-plugin-priorities
    -   rpm -Uvh <https://repo.opensciencegrid.org/osg/3.3/osg-3.3-el7-release-latest.rpm>
    -   yum install osg-oasis 
2. Verify latest versions of cvmfs, cvmfs-config-osg, and cvmfs-x509-helper have been installed. 
3. Edit fuse configuration.
    -   vi /etc/fuse.conf
        -   user_allow_other 
4. Edit cvmfs configuration to point to replica server.
    -   vi /etc/cvmfs/default.local
        -   CVMFS_REPOSITORIES="\`echo $((echo oasis.opensciencegrid.org;echo cms.cern.ch;ls /cvmfs)|sort -u)|tr ' ' ,\`"
        -   CVMFS_QUOTA_LIMIT=20000
        -   CVMFS_HTTP_PROXY=DIRECT
5. Edit autofs configuration and restart service.
    -   vi /etc/auto.master
        -   /cvmfs /etc/auto.cvmfs
    -   service autofs restart
6. Run CVMFS config probe.
    -   cvmfs_config probe 
7. Check CVMFS config status.
    -   cvmfs_config stat -v oasis.opensciencegrid.org

### Additional Documentation ###

-   [CERN's CVMFS Documentation](https://cvmfs.readthedocs.io/en/stable/)
-   [OSG's CVMFS Replica Server](https://www.opensciencegrid.org/docs/other/install-cvmfs-stratum1/)
-   [OSG's CVMFS Client Documentation](https://www.opensciencegrid.org/docs/worker-node/install-cvmfs/)
-   [OSG's OASIS Documentation](https://www.opensciencegrid.org/docs/data/external-oasis-repos/)

-   [bash_pi.sh](bash_pi.sh): A bash script that uses a simple Monte Carlo method to estimate the value of Pi


Gratia Probe
------------

This section documents the testing procedure to test the gratia probes sufficiently tested to be promoted to the osg-testing repository. The test procedure is the same on both SL6 and SL7.

-   install or update the `gratia-probe-condor` rpm as appropriate
-   On each VM download the gratia-probe-setup.sh script and run it
-   In `/etc/gratia/condor/ProbeConfig`, verify the following have been changed:
    -   change `SiteName` to something aside from `Generic Site`
    -   change `EnableProbe` to 1
    -   change `CollectorHost`, `SSLHost`, and `SSLRegistrationHost` to the an invalid host (E.g. test.com) or the localhost
-   Create `/var/lib/osg/` and download the attached `user-vo-map` file and place it in that directory
-   Edit the `user-vo-map` file and change the account from `sthapa` to the account you'll be using to submit the condor jobs in the following step
-   Download and submit the attached condor\_submit file (note, on the default fermicloud VM, this takes about 3 hours, so you may want to set `NUM_CPUS` to 50 so that 50 jobs will run at a time)
-   Run `/usr/share/gratia/condor/condor_meter`
-   Check `/var/lib/gratia/tmp/gratiafiles/` for a `subdir.condor_...` directory and verify that there are 200 xml jobs and the cpus/wall times are appropriate (either PT0S or PT1M).


GSI OpenSSH
-----------

To test a fresh installation:

1.  Spin up two VM's and set up the EPEL/OSG repos on both of them.
2.  Choose one of the VM's, it will be the server VM. Consult these [instructions](https://www.opensciencegrid.org/docs/other/gsissh/) to set up the server.
3.  From the other VM (client):
    1.   Install the necessary packages:
    
            :::console
            root@host # yum install globus-proxy-utils gsi-openssh-clients

    1.   Initialize your proxy. After this, none of the gsi commands should prompt you for your password.
    1.   Connect to the server:
    
            :::console
            user@host $ gsissh -p 2222 %RED%<server hostname>%ENDCOLOR%

    1.   Copy a test file to the server:
    
            :::console
            user@host $ gsiscp -p 2222 testfile %RED%<server hostname>%ENDCOLOR%:/tmp

    1.   Connect to the server via SFTP and grab files:
    
            :::console
            user@host $ gsisftp -P 2222 %RED%<server hostname>%ENDCOLOR%
            user@host $ cd /tmp
            user@host $ get testfile

HDFS (WIP)
----------

That was it. New instructions:

I set up a namenode, datanode, and gridftp on the same SLF6 VM. I did:
- edit {{/etc/hadoop/conf/hdfs-site.xml}}
-- set {{dfs.replication}} to 1
-- set {{dfs.replication.min}} to 1
- {noformat}
hadoop fs -mkdir /matyas
hadoop fs -chown matyas /matyas
hdfs dfsadmin -setSpaceQuota 123k /matyas
{noformat}
As matyas:
{noformat}
dd if=/dev/zero of=/tmp/blob bs=4096 count=10000
kx509; voms-proxy-init -noregen -voms fermilab
globus-url-copy -vb file:///tmp/blob gsiftp://`hostname -f`/mnt/hadoop/matyas
{noformat}

In the pre-patch version, the error was "Unknown error 255". In the post-patch version, the error was "Disk quota exceeded".

I repeated the test on an SLF7 VM and got exactly the same results.

Hadoop is not covered by osg-test, but the above commands should verify basic functionality (ls, mkdir, chown). With a sufficiently large quota, file transfers via {{globus-url-copy}} continued to work as expected. (Note: the quota must not only be greater than the file size, but greater than the amount of space HDFS consumes storing the file. In my case, the test file was ~40MB, but I needed to increase the quota above 256MB).

HTCondor-CE Collector (WIP)
---------------------------

The CE Collector is a stripped-down version of HTCondor-CE that contains mostly just the collector daemon and configs. It was introduced in htcondor-ce-1.6. The production CE Collectors run at the GOC, but you may want to set up your own for testing.

1.  Make 2 VMs with the EPEL/OSG repos installed: one for the collector, and one for the CE

### Setting Up the Collector

1.  Install `htcondor-ce-collector`
2.  Create a file called `/etc/condor-ce/config.d/99-local.conf` that contains this line:<pre class='file'>

        COLLECTOR.ALLOW_ADVERTISE_SCHEDD = $(COLLECTOR.ALLOW_ADVERTISE_SCHEDD), your_htcondor_ce_host.example.net</pre>

    (with `your_htcondor_ce_host` replaced by the hostname the HTCondor-CE VM)

3.  Run `service condor-ce-collector start`

### Setting Up the CE

1.  Install `osg-htcondor-ce-condor` (replace condor with the batch system of your choice)
2.  Ensure osg-configure >= 1.0.60-2 is installed
3.  Configure your CE using osg-configure
    1.  You should use the [HTCondor-CE Install Docs](https://www.opensciencegrid.org/docs/compute-element/install-htcondor-ce/) as a reference, although you can skip several of the steps
    2.  You can skip setting up Squid: set `enabled` to `True` and `location` to `UNAVAILABLE` in `01-squid.ini`
    3.  Set `htcondor_gateway_enabled` to `True` in `10-gateway.ini`
    4.  You probably don't need GUMS, but if you want it, use the Fermi GUMS server (set `gums_host` to `gums.fnal.gov` and `authorization_method` to `xacml` in 10-misc.ini)
    5.  To keep osg-configure from complaining about storage, edit `10-storage.ini`:
        -   Set `se_available` to `False`
        -   Set `app_dir` to `/osg/app_dir`
        -   Set `data_dir` to `/osg/data_dir`
        -   Do `mkdir -p /osg/app_dir/etc; mkdir -p /osg/data_dir; chmod 1777 /osg/app_dir{,/etc}`

    6.  Enable your batch system by setting `enabled` to `True` in `20-<batch system>.ini`
    7.  Set up the site info in `40-siteinfo.ini` ; in particular, you'll need to set the `resource` and `resource_group` settings
        \\ (you just need to pick a name; I concatenate my login name with the short host name and use that, e.g. matyasfermicloud001).
        \\ You can also use the following settings:
        -   `group=OSG-ITB`
        -   `sponsor=local`
        -   `city=Batavia, IL`
        -   `country=US`
        -   `longitude=-88`
        -   `latitude=41`

4.  Edit the file `/etc/osg/config.d/30-infoservices.ini` and set `ce_collectors` to the collector host
5.  Run `osg-configure -dc`
6.  Start up your batch system
7.  Run `service condor-ce start`

The CE will report to the collector host every five minutes. If you want to force it to send now, run `condor_ce_reconfig`. You should see your CE if you run `condor_ce_status -schedd` on the collector host.

RSV
---

Testing a fresh installation:

1.  make sure the yum repositories required by OSG is installed on your host
    -   rpm -Uvh <http://repo.opensciencegrid.org/osg/3.4/osg-3.4-el7-release-latest.rpm> OR rpm -Uvh <http://repo.opensciencegrid.org/osg/3.4/osg-3.4-el6-release-latest.rpm>
    -   also make sure epel repo is set up. 
2. install the rpm
    -   yum --enablerepo=osg-testing install rsv 
3. edit /etc/osg/config.d/30-rsv.ini file
    -   in my case, I don't have a service cert for testing, so I use my own personal cert to create the proxy, but later on the owner of the proxy should be changed to "rsv" user that is created during the rpm install.
    -   in the config file, for the ce\_hosts and gridftp\_hosts, put in a test server, as the result from this test will be uploaded to OSG GOC, which may mess up your production service monitoring if you chose a production server for the test.
4. osg-configure -v
5. osg-configure -c
6. /etc/init.d/condor-cron start
7. /etc/init.d/rsv start
8. rsv-control --list
9. rsv-control --version
10. rsv-control --run --all-enabled 11. make sure the results from the above commands look fine.

Testing an upgrade installation:

1.  make sure to enable the osg-testing repo, and set its priority higher than the other repos
2. yum --enablerepo=osg-testing upgrade rsv\*
3. you can use the old 30-rsv.ini file for configuration
4. repeat steps 4)~11) as mentioned in the last section.


Slurm
-----

This section goes through the steps needed to set up a slurm install on a VM. This is a necessary prerequisite for testing Slurm related components (CE integration, gratia, etc.). Note that the slurm setup used for this uses weak passwords for mysql. It should be sufficient for a quick setup, testing, and then tear down but should not be used without changes if it will be running for any appreciable length of time.

!!! note
    need to have a VM with 2+ GB of memory

### Installation and setup ###

1. Download scripts and config files: 

        cd /tmp/
        git clone <https://github.com/sthapa/utilities.git>
        cd utilities/slurm 

2. setup and install slurm components

        export username='USERNAME' \# user that jobs will run as
        export version='14.11.7' \# slurm version to install (e.g. 16.05.2 or 14.11.7)
        ./slurm-setup.sh 

3. After successful completion, slurm and slurm gratia probes should be setup and enabled.

### Running a job using slurm ###

1.  Generate test.sh with the following:

        #/bin/bash
        echo "In the directory: `pwd`" echo "As the user: `whoami`" echo “Hostname:" /bin/hostname sleep 60 </pre> 

2. run `sbatch test.sh`
3. the output from the jobs should appear in the current working directory as `test.sh.[eo].nnnnn` where nnnnn is a job id

VO Client
---------

This document contains a basic recipe for testing a VO Package release

### Prerequisites ###

Testing the VO package requires a few components:
   * X.509 certificate with membership to at least one VO
   * System with working GUMS installation
   * System with OSG installation (voms-proxy-init and edg-mkgridmap)
   

### Testing `voms-proxy-init` ###

Login in the system that has voms-proxy-init installed.  Make sure that you have the correct 
vo-client rpms installed and that your X.509 certificate is in your home directory.  For each
VO that you have membership in, run the following `voms-proxy-init -voms [VO]` where `[VO]` is 
the appropriate VO (e.g. osg, cms, etc.).  You should be able to generate a voms-proxy for that
VO without errors.


### Testing `edg-mkgridmap` ###

Log on to a system with `edg-mkgridmap` installed.  Make sure you have the correct vo-client rpms 
installed (vo-client-edgmkgridmap).  Run `edg-mkgridmap` and check the log output for errors.  
There will be some errors so compare your errors with the errors on previous vo-package tickets to
make sure no new errors have appeared.

### Testing GUMS ###

Log on to a system with a working GUMS install.  Make sure that you have the correct vo-client 
rpms (osg-gums-config) installed.  


1. Make a backup of `/etc/gums/gums.config` 
1. Copy the mysql database information from `/etc/gums/gums.config` to `/etc/gums/gums.config.template`
1. Copy `/etc/gums/gums.config.template` to `/etc/gums/gums.config`
1. Start the `tomcat6` service 
1. Go to the GUMS interface (e.g. https://my.host:8443/gums)
1. Go to the Update VO members page and click on the `update VO members` button
1. Once completed, there will probably be some errors.
1. Compare errors to errors on prior vo package update tickets and make sure no new errors have occurred. 

VOMS Admin Server
-----------------

### Install and configure voms-admin-server ###

Install and configure voms-admin-server with this [voms-install.sh](voms-install.sh) script, entering `osg-testing` when prompted for the `REPO` and your own e-mail address when prompted for `EMAIL_FROM`.

### Set up TEST_VO and add yourself as an admin ###

To add a test VO (named TEST\_VO) and add yourself as an admin, use the following script, replacing the `USER_EMAIL`, `USER_CERT_SUBJECT`, and `USER_COMMON_NAME` variables with your own:

``` file
#!/bin/bash

VO_NAME="TEST_VO"
TOMCAT_PORT="8443"
USER_EMAIL="YOUR_EMAIL"
USER_CERT_SUBJECT="YOUR CERT DN"
USER_CERT_ISSUER="/DC=com/DC=DigiCert-Grid/O=DigiCert Grid/CN=DigiCert Grid CA-1"
USER_COMMON_NAME="YOUR CERT CN"

voms-admin --vo $VO_NAME --host $HOSTNAME --nousercert create-user \
    "$USER_CERT_SUBJECT" "$USER_CERT_ISSUER" "$USER_COMMON_NAME" "$USER_EMAIL"

voms-admin --vo $VO_NAME --host $HOSTNAME --nousercert assign-role \
    /$VO_NAME VO-Admin "$USER_CERT_SUBJECT" "$USER_CERT_ISSUER"

echo web ui: https://$HOSTNAME:$TOMCAT_PORT/voms/$VO_NAME
```

### Testing the web UI ###

1.  Add/suspend/restore/delete a user (using your e-mail address for contact)
2.  Verify e-mail receipt of suspension message

XRootD VOMS Testing
-------------------

This section is intended for OSG Software/Release teams to gather information on testing vomsxrd/xrootd-voms-plugin package. Original plugin named [vomsxrd](http://gganis.github.io/vomsxrd/), similar to lcmaps that extracts information for authorization within xrootd of a proxy's voms extension.

You need an [xrootd server installation](https://www.opensciencegrid.org/docs/data/install-xrootd/)

In the xrootd server yum install the following packages:

-   xrootd
-   xrootd-voms-plugin
-   vo-client

In the xrootd client yum install the following packages:

-   xrootd-client
-   voms-clients
-   vo-client

In the xrootd server add this lines to file `/etc/xrootd/xrootd-clustered.cfg`

    xrootd.seclib /usr/lib64/libXrdSec.so
    sec.protparm gsi -vomsfun:/usr/lib64/libXrdSecgsiVOMS.so -vomsfunparms:certfmt=raw|vos=cms|dbg -vomsat:2
    sec.protocol /usr/lib64 gsi -ca:1 -crl:3

This configuration will only authorize members of VO **cms**. You can change it to another VO.

Make sure `fetch-crl` has been run otherwise the xrootd service may fail to start.

In the xrootd client get a proxy without voms extension or with another VO extension different that the one used in the configuration:

``` console
user@host $ voms-proxy-init -voms mis
Enter GRID pass phrase:
Your identity: /DC=com/DC=DigiCert-Grid/O=Open Science Grid/OU=People/CN=Edgar Mauricio Fajardo Hernandez 2020
Creating temporary proxy ........................... Done
Contacting  voms.opensciencegrid.org:15001 [/DC=com/DC=DigiCert-Grid/O=Open Science Grid/OU=Services/CN=http/voms.opensciencegrid.org] "mis" Done
Creating proxy ............................................... Done
user@host $ xrdcp vomsxrdtest root://fermicloud024.fnal.gov:1094//tmp/
[0B/0B][100%][==================================================][0B/s]  
Run: [FATAL] Auth failed
```

Now get a proxy with cms extension and run it again:

``` console
user@host $ voms-proxy-init -voms cms
Enter GRID pass phrase:
Your identity: /DC=com/DC=DigiCert-Grid/O=Open Science Grid/OU=People/CN=Edgar Mauricio Fajardo Hernandez 2020
Creating temporary proxy ...................................... Done
Contacting  voms2.cern.ch:15002 [/DC=ch/DC=cern/OU=computers/CN=voms2.cern.ch] "cms" Done
Creating proxy .......................................... Done
Your proxy is valid until Thu Dec  4 22:53:29 2014
user@host $ xrdcp vomsxrdtest root://fermicloud024.fnal.gov:1094//tmp/
[0B/0B][100%][==================================================][0B/s] 
```


XRootD Plugins
--------------

### Hadoop name node installation ###

Use the following script with option 1:

``` file
#!/bin/bash
set -e

select NODETYPE in namenode datanode gridftp; do
  [[ $NODETYPE ]] && break
done
case $NODETYPE in
  namenode ) NAMENODE=$HOSTNAME ;;
         * ) read -p 'hostname for NAMENODE? ' NAMENODE ;;
esac
echo NODETYPE=$NODETYPE
echo NAMENODE=$NAMENODE
read -p 'ok? [y/N] ' ok
case $ok in
  y*|Y*) ;;  # ok
      *) exit ;;
esac
#yum install --enablerepo=osg-minefield osg-se-hadoop-$NODETYPE
yum install osg-se-hadoop-$NODETYPE
case $NODETYPE in
  namenode|datanode )
    mkdir -p /data/{hadoop,scratch,checkpoint}
    chown -R hdfs:hdfs /data
    sed -i s/NAMENODE/$NAMENODE/ /etc/hadoop/conf.osg/{core,hdfs}-site.xml
    cp /etc/hadoop/conf.osg/{core,hdfs}-site.xml /etc/hadoop/conf/
    touch /etc/hosts_exclude
    ;;
  gridftp )
    ln -snf conf.osg /etc/hadoop/conf
    sed -i s/NAMENODE/$NAMENODE/ /etc/hadoop/conf.osg/{core,hdfs}-site.xml
    echo "hadoop-fuse-dfs# /mnt/hadoop fuse server=$NAMENODE,port=9000,rdbuffer=131072,allow_other 0 0" >> /etc/fstab
    mkdir /mnt/hadoop
    mount /mnt/hadoop
    cp -v /etc/redhat-release /mnt/hadoop/test-file
    sed -i '/globus_mapping/s/^# *//' /etc/grid-security/gsi-authz.conf
    sed -i s/yourgums.yourdomain/gums.fnal.gov/ /etc/lcmaps.db
    mkdir /mnt/hadoop/fnalgrid
    useradd fnalgrid -g fnalgrid
    chown fnalgrid:fnalgrid /mnt/hadoop/fnalgrid
    service globus-gridftp-server start
    if type -t globus-url-copy >/dev/null; then
      globus-url-copy file:////bin/bash  gsiftp://$HOSTNAME/mnt/hadoop/fnalgrid/first_test
    else
      echo globus-url-copy not installed
    fi
    ;;
esac
case $NODETYPE in
  namenode ) su - hdfs -c "hadoop namenode -format" ;;
esac
service hadoop-hdfs-$NODETYPE start
```

### Hadoop data node installation ###

1.  Run same script as before but with option number 2.
1.  Install xrootd-server:

        yum install xrootd-server

1.  Install xrootd-plugins

        yum install xrootd-cmstfc xrootd-hdfs

### GridFTP installation ###

Run same as script but with option number.

### On the name node ###

``` console
[root@fermicloud092 ~]# hdfs dfs -ls /test-file
Found 1 items
-rw-r--r--   2 root root          0 2014-07-21 15:57 /test-file
```

This means your hadoop is working.

Modify the file `/etc/xrootd/xrootd-clustered.cfg` to look like this:

``` file
xrd.port 1094

# The roles this server will play.                                                                                            
all.role server
all.role manager if xrootd.unl.edu
# The known managers                                                                                                          
all.manager srm.unl.edu:1213
#all.manager xrootd.ultralight.org:1213                                                                                       

# Allow any path to be exported; this is further refined in the authfile.                                                     
all.export / nostage

# Hosts allowed to use this xrootd cluster                                                                                    
cms.allow host *

### Standard directives                                                                                                       
# Simple sites probably don't need to touch these.                                                                            
# Logging verbosity                                                                                                           
xrootd.trace all -debug
ofs.trace all -debug
xrd.trace all -debug
cms.trace all -debug

# Integrate with CMS TFC, placed in /etc/storage.xml                                                                          
oss.namelib /usr/lib64/libXrdCmsTfc.so file:/etc/xrootd/storage.xml?protocol=hadoop

xrootd.seclib /usr/lib64/libXrdSec.so
xrootd.fslib /usr/lib64/libXrdOfs.so
ofs.osslib /usr/lib64/libXrdHdfs.so
all.adminpath /var/run/xrootd
all.pidpath /var/run/xrootd

cms.delay startup 10
cms.fxhold 60s
cms.perf int 30s pgm /usr/bin/XrdOlbMonPerf 30

oss.space default_stage /opt/xrootd_cache
```

Create file `/etc/xrootd/storage.xml` and place this:

    <storage-mapping>
    <lfn-to-pfn protocol="hadoop" destination-match=".*" path-match=".*/+tmp2/test-file" result="/test-file"/>
    </storage-mapping>

For el7 the instrucctions are a little bit different. See:

<https://jira.opensciencegrid.org/browse/SOFTWARE-2198?focusedCommentId=334667&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-334667>

Now from a node do:

``` console
xrdcp --debug 3 root://yourdatanode.yourdomain:1094//tmp2/test-file .
```

If it is sucessful it would have tested both cmstfc and hdfs plugins 
