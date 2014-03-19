Notes on Installing vsftpd from Scratch in Ubuntu 12.04
=======================================================

I. Errors that may occur during compilation
---------------------

__Issue 1.1__

Make sure that you have `libpam0g-dev` installed on your machine. If not, even you have successfully built a vsftpd binary file, it is not linked to `libpam`, which will eventually result to login problems.

*Solution:*

    apt-get install libpam0g-dev

__Issue 1.2__

If `libpam0g-dev` is installed:

    sysdeputil.o: In function `vsf_sysdep_check_auth':
    sysdeputil.c:(.text+0xeca): undefined reference to `pam_start'
    sysdeputil.c:(.text+0xef4): undefined reference to `pam_set_item'
    sysdeputil.c:(.text+0xf1c): undefined reference to `pam_set_item'
    sysdeputil.c:(.text+0xf46): undefined reference to `pam_set_item'
    sysdeputil.c:(.text+0xf64): undefined reference to `pam_authenticate'
    sysdeputil.c:(.text+0xf8a): undefined reference to `pam_get_item'
    sysdeputil.c:(.text+0xfb8): undefined reference to `pam_acct_mgmt'
    sysdeputil.c:(.text+0xfd6): undefined reference to `pam_setcred'
    sysdeputil.c:(.text+0x1010): undefined reference to `pam_open_session'
    sysdeputil.c:(.text+0x1046): undefined reference to `pam_end'
    sysdeputil.c:(.text+0x107e): undefined reference to `pam_end'
    sysdeputil.c:(.text+0x109e): undefined reference to `pam_end'
    sysdeputil.c:(.text+0x10b6): undefined reference to `pam_end'
    sysdeputil.c:(.text+0x10e2): undefined reference to `pam_setcred'
    sysdeputil.o: In function `vsf_auth_shutdown':
    sysdeputil.c:(.text+0x1115): undefined reference to `pam_close_session'
    sysdeputil.c:(.text+0x112b): undefined reference to `pam_setcred'
    sysdeputil.c:(.text+0x1141): undefined reference to `pam_end'
    collect2: ld returned 1 exit status
    make: *** [vsftpd] Error 1

*Solution:*

Modify `LIBS` arg in `Makefile` to
    
    LIBS = `./vsf_findlibs.sh` -lpam

__Issue 2__

    tcpwrap.c:16:20: fatal error: tcpd.h: No such file or directory
    compilation terminated.

*Solution:*

    apt-get install libwrap0 libwrap0-dev
    
__Issue 3__

    sysdeputil.o: In function `vsf_sysdep_check_auth':
    sysdeputil.c:(.text+0x109): undefined reference to `crypt'
    sysdeputil.c:(.text+0x13a): undefined reference to `crypt'
    collect2: ld returned 1 exit status
    make: *** [vsftpd] Error 1

*Solution:*

Modify `LIBS` arg in `Makefile` to

    LIBS = `./vsf_findlibs.sh` -lcrypt

If you already append `-lpam` to it, it should be like:

    LIBS = `./vsf_findlibs.sh` -lcrypt -lpam


II. Configuration
-----------------

Some directories and files used in `vsftpd.conf` may vary from that are given in INSTALL file or examples, and they should be changed accordingly.

__1. `xinetd`__

  If you prefer running vsftpd in xinetd mode, once the compilation is done, do `apt-get install xinetd` first before typing `make install`.
  And change `server_args` in `/etc/xinetd.d/vsftpd` to the location where you put `vsftpd.conf`:
  
    server_args = /etc/vsftpd/vsftpd.conf

__2. Virtual Users__
   
  In Ubuntu 12.04, `pam_userdb.so` is located in `/lib/i386-linux-gnu/security`. Thus `path/to/pam_userdb.so` in pam file should be changed. eg.:
  
    auth required /lib/i386-linux-gnu/security/pam_userdb.so db=/etc/vsftpd/vsftpd_login
    account required /lib/i386-linux-gnu/security/pam_userdb.so db=/etc/vsftpd/vsftpd_login
  
  But actually if only filenames are given it just does fine:
  
    auth required pam_userdb.so db=/etc/vsftpd/vsftpd_login
    account required pam_userdb.so db=/etc/vsftpd/vsftpd_login
    session required pam_loginuid.so

III. Misc.
----------

__1. `ldd vsftpd`__

  Do `ldd vsftpd` once you have done compiling to make sure vsftpd is built with libraries you need.
  If you enabled `VSF_BUILD_TCPWRAPPERS`, `VSF_BUILD_PAM`, `VSF_BUILD_SSL` in `builddefs.h`, `ldd` probably should return:
  
    linux-gate.so.1 =>  (0xb76f1000)
    libwrap.so.0 => /lib/i386-linux-gnu/libwrap.so.0 (0xb76a9000)
    libssl.so.1.0.0 => /lib/i386-linux-gnu/libssl.so.1.0.0 (0xb7651000)
    libcrypto.so.1.0.0 => /lib/i386-linux-gnu/libcrypto.so.1.0.0 (0xb74a5000)
    libpam.so.0 => /lib/i386-linux-gnu/libpam.so.0 (0xb7497000)
    libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb72ed000)
    libnsl.so.1 => /lib/i386-linux-gnu/libnsl.so.1 (0xb72d3000)
    libdl.so.2 => /lib/i386-linux-gnu/libdl.so.2 (0xb72ce000)
    libz.so.1 => /lib/i386-linux-gnu/libz.so.1 (0xb72b7000)
    /lib/ld-linux.so.2 (0xb76f2000)
    
__2. Configuration guide__

    A configuration guide can be found at <https://help.ubuntu.com/community/vsftpd>.





    