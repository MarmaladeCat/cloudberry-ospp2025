#-------------------------------------------------------------------------
#
# Makefile for the postgres backend
#
# Portions Copyright (c) 1996-2021, PostgreSQL Global Development Group
# Portions Copyright (c) 1994, Regents of the University of California
#
# src/backend/Makefile
#
#-------------------------------------------------------------------------

PGFILEDESC = "PostgreSQL Server"
# This is a daemon process, which is why it is not labeled as an executable
#PGAPPICON=win32

subdir = src/backend
top_builddir = ../..
include $(top_builddir)/src/Makefile.global

SUBDIRS = access bootstrap catalog parser commands executor foreign lib libpq \
	main nodes optimizer partitioning port postmaster \
	regex replication rewrite \
	statistics storage tcop tsearch utils $(top_builddir)/src/timezone \
	jit task crypto

SUBDIRS += fts cdb
ifeq ($(enable_orca),yes)
SUBDIRS += gporca gpopt
endif
ifeq ($(enable_catalog_ext),yes)
SUBDIRS += catalog-extension
LDFLAGS += -lprotobuf -lstdc++
endif

include $(srcdir)/common.mk

# As of 1/2010:
# The probes.o file is necessary for dtrace support on Solaris, and on recent
# versions of systemtap.  (Older systemtap releases just produce an empty
# file, but that's okay.)  However, macOS's dtrace doesn't use it and doesn't
# even recognize the -G option.  So, build probes.o except on macOS.
# This might need adjustment as other platforms add dtrace support.
ifneq ($(PORTNAME), darwin)
ifeq ($(enable_dtrace), yes)
LOCALOBJS += utils/probes.o
endif
endif

# FIXME: The --enable-shared-postgres-backend parameter build the libpostgres.so shared between postmaster
# But we only test it locally without CI test. Please use it with caution.
OBJS = \
	$(LOCALOBJS) \
	$(SUBDIROBJS) \
	$(top_builddir)/src/common/libpgcommon_srv.a \
	$(top_builddir)/src/port/libpgport_srv.a

ifeq ($(PORTNAME), darwin)
LDOPTS = -Z
endif

# We put libpgport and libpgcommon into OBJS, so remove it from LIBS; also add
# libldap and ICU
LIBS := $(filter-out -lpgport -lpgcommon, $(LIBS)) $(LDAP_LIBS_BE) $(ICU_LIBS)

# The backend doesn't need everything that's in LIBS, however
LIBS := $(filter-out -lreadline -ledit -ltermcap -lncurses -lcurses, $(LIBS))

# Cloudberry uses threads in the backend
LIBS := $(LIBS) -lpthread

ifeq ($(with_systemd),yes)
LIBS += -lsystemd
endif

##########################################################################

all: submake-libpgport submake-catalog-headers submake-utils-headers postgres $(POSTGRES_IMP)

ifneq ($(PORTNAME), cygwin)
ifneq ($(PORTNAME), win32)
ifneq ($(PORTNAME), aix)

# Due to Cloudberry's MPP architecture, the postgres process needs to connect to
# databases, some symbols from frontend are compiled to the postgres binary.
# Some extensions requires frontend symbols and are dynamically linked to libpq.so.
# However, some symbols are not safe to be exported both in the postgres binary
# and libpq.so. E.g., some of the functions are using palloc()/pfree() to allocate
# and free memory in backend codes while using malloc()/free() to allocate and free
# memory in frontend codes. We use linker version script to make these symbols exported
# in libpq.so and private in the postgres binary.
# FIXME: Can we teach the postgres binary to dynamically link against libpq.so? So that
# we don't need this hack.
SAFELY_EXPORTED_SYMBOLS_PATTERN = "(pqsignal|pg_*)"
SYMBOL_MAPPING_FLAGS =
ifeq ($(PORTNAME), darwin)
SYMBOL_MAP_FILE = hide_symbols.list
$(SYMBOL_MAP_FILE): $(top_builddir)/src/interfaces/libpq/exports.txt
	$(AWK) '/^[^\#]/ {printf "_%s\n",$$1}' $< | grep -v -E $(SAFELY_EXPORTED_SYMBOLS_PATTERN) >$@
SYMBOL_MAPPING_FLAGS = -unexported_symbols_list $(SYMBOL_MAP_FILE)
endif

ifeq ($(PORTNAME), linux)
SYMBOL_MAP_FILE = postgres_symbols.map
$(SYMBOL_MAP_FILE): $(top_builddir)/src/interfaces/libpq/exports.txt
	( echo '{ global: *; '; echo ' local:'; $(AWK) '/^[^\#]/ {printf "%s;\n",$$1}' $< | grep -v -E $(SAFELY_EXPORTED_SYMBOLS_PATTERN); echo '};' ) >$@
SYMBOL_MAPPING_FLAGS = -Wl,--version-script=$(SYMBOL_MAP_FILE)
endif
ifeq ($(enable_shared_postgres_backend),yes)
all: libpostgres.so
libpostgres.so: $(OBJS) $(SYMBOL_MAP_FILE)
	$(CXX) -shared $(CXXFLAGS) $(LDFLAGS) $(LDFLAGS_SL) $(export_dynamic) \
	  $(filter-out main/main.o, $(call expand_subsys,$(OBJS))) $(LIBS) $(SYMBOL_MAPPING_FLAGS) -o $@

# if enable-build-postgres-with-shared is yes, link postgres with shared library libpostgres.so
ifeq ($(enable_link_postgres_with_shared),yes)
postgres: main/main.o libpostgres.so $(top_builddir)/src/port/libpgport_srv.a $(top_builddir)/src/common/libpgcommon_srv.a
	$(CXX) $(CXXFLAGS) $(LDFLAGS) $(LDFLAGS_EX) $(export_dynamic) \
	  main/main.o libpostgres.so $(top_builddir)/src/port/libpgport_srv.a \
	  $(top_builddir)/src/common/libpgcommon_srv.a $(LIBS) -o $@
else
postgres: $(OBJS) $(SYMBOL_MAP_FILE)
	$(CXX) $(CXXFLAGS) $(call expand_subsys,$(OBJS)) $(LDFLAGS) $(LDFLAGS_EX) $(export_dynamic) $(LIBS) $(SYMBOL_MAPPING_FLAGS) -o $@
endif

else
postgres: $(OBJS) $(SYMBOL_MAP_FILE)
	$(CXX) $(CXXFLAGS) $(call expand_subsys,$(OBJS)) $(LDFLAGS) $(LDFLAGS_EX) $(export_dynamic) $(LIBS) $(SYMBOL_MAPPING_FLAGS) -o $@
endif

endif
endif
endif

ifeq ($(PORTNAME), cygwin)

postgres: $(OBJS)
	$(CXX) $(CXXFLAGS) $(call expand_subsys,$^) $(LDFLAGS) $(LDFLAGS_EX) $(export_dynamic) -Wl,--stack,$(WIN32_STACK_RLIMIT) -Wl,--export-all-symbols -Wl,--out-implib=libpostgres.a $(LIBS) -o $@

# libpostgres.a is actually built in the preceding rule, but we need this to
# ensure it's newer than postgres; see notes in src/backend/parser/Makefile
libpostgres.a: postgres
	touch $@

endif # cygwin

ifeq ($(PORTNAME), win32)
LIBS += -lsecur32

postgres: $(OBJS) $(WIN32RES)
	$(CXX) $(CXXFLAGS) $(call expand_subsys,$(OBJS)) $(WIN32RES) $(LDFLAGS) $(LDFLAGS_EX) -Wl,--stack=$(WIN32_STACK_RLIMIT) -Wl,--export-all-symbols -Wl,--out-implib=libpostgres.a $(LIBS) -o $@$(X)

# libpostgres.a is actually built in the preceding rule, but we need this to
# ensure it's newer than postgres; see notes in src/backend/parser/Makefile
libpostgres.a: postgres
	touch $@

endif # win32

ifeq ($(PORTNAME), aix)

postgres: $(POSTGRES_IMP)
	$(CXX) $(CXXFLAGS) $(call expand_subsys,$(OBJS)) $(LDFLAGS) $(LDFLAGS_EX) -Wl,-bE:$(top_builddir)/src/backend/$(POSTGRES_IMP) $(LIBS) -Wl,-brtllib -o $@

$(POSTGRES_IMP): $(OBJS)
	$(LD) $(LDREL) $(LDOUT) SUBSYS.o $(call expand_subsys,$^)
ifeq ($(host_os), aix3.2.5)
	$(MKLDEXPORT) SUBSYS.o $(bindir)/postgres > $@
else
ifneq (,$(findstring aix4.1, $(host_os)))
	$(MKLDEXPORT) SUBSYS.o $(bindir)/postgres > $@
else
	$(MKLDEXPORT) SUBSYS.o . > $@
endif
endif
	@rm -f SUBSYS.o

endif # aix

$(top_builddir)/src/port/libpgport_srv.a: | submake-libpgport


# The postgres.o target is needed by the rule in Makefile.global that
# creates the exports file when MAKE_EXPORTS = true.
postgres.o: $(OBJS)
	$(CXX) $(LDREL) $(call expand_subsys,$^) $(LDFLAGS) $(LDFLAGS_EX) $(LIBS) -o $@

# The following targets are specified in make commands that appear in
# the make files in our subdirectories. Note that it's important we
# match the dependencies shown in the subdirectory makefiles!
# Also, in cases where a subdirectory makefile generates two files in
# what's really one step, such as bison producing both gram.h and gram.c,
# we must request making the one that is shown as the secondary (dependent)
# output, else the timestamp on it might be wrong.  By project convention,
# the .h file is the dependent one for bison output, so we need only request
# that; but in other cases, request both for safety.

parser/gram.h: parser/gram.y
	$(MAKE) -C parser gram.h

storage/lmgr/lwlocknames.h: storage/lmgr/generate-lwlocknames.pl storage/lmgr/lwlocknames.txt
	$(MAKE) -C storage/lmgr lwlocknames.h lwlocknames.c

# run this unconditionally to avoid needing to know its dependencies here:
submake-catalog-headers:
	$(MAKE) -C catalog distprep generated-header-symlinks

# run this unconditionally to avoid needing to know its dependencies here:
submake-utils-headers:
	$(MAKE) -C utils distprep generated-header-symlinks

.PHONY: submake-catalog-headers submake-utils-headers

# Make symlinks for these headers in the include directory. That way
# we can cut down on the -I options. Also, a symlink is automatically
# up to date when we update the base file.
#
# The point of the prereqdir incantation in some of the rules below is to
# force the symlink to use an absolute path rather than a relative path.
# For headers which are generated by make distprep, the actual header within
# src/backend will be in the source tree, while the symlink in src/include
# will be in the build tree, so a simple ../.. reference won't work.
# For headers generated during regular builds, we prefer a relative symlink.

.PHONY: generated-headers

generated-headers: $(top_builddir)/src/include/parser/gram.h $(top_builddir)/src/include/storage/lwlocknames.h submake-catalog-headers submake-utils-headers

$(top_builddir)/src/include/parser/gram.h: parser/gram.h
	prereqdir=`cd '$(dir $<)' >/dev/null && pwd` && \
	  cd '$(dir $@)' && rm -f $(notdir $@) && \
	  $(LN_S) "$$prereqdir/$(notdir $<)" .

$(top_builddir)/src/include/storage/lwlocknames.h: storage/lmgr/lwlocknames.h
	prereqdir=`cd '$(dir $<)' >/dev/null && pwd` && \
	  cd '$(dir $@)' && rm -f $(notdir $@) && \
	  $(LN_S) "$$prereqdir/$(notdir $<)" .

utils/probes.o: utils/probes.d $(SUBDIROBJS)
	$(DTRACE) $(DTRACEFLAGS) -C -G -s $(call expand_subsys,$^) -o $@


##########################################################################

# Be sure that these files get removed by the maintainer-clean target
distprep:
	$(MAKE) -C parser	gram.c gram.h scan.c
	$(MAKE) -C bootstrap	bootparse.c bootscanner.c
	$(MAKE) -C catalog	distprep
	$(MAKE) -C replication	repl_gram.c repl_scanner.c syncrep_gram.c syncrep_scanner.c
	$(MAKE) -C storage/lmgr	lwlocknames.h lwlocknames.c
	$(MAKE) -C utils	distprep
	$(MAKE) -C utils/adt	jsonpath_gram.c jsonpath_scan.c
	$(MAKE) -C utils/misc	guc-file.c


##########################################################################

install: all installdirs install-bin
ifeq ($(PORTNAME), cygwin)
ifeq ($(MAKE_DLL), true)
	$(INSTALL_DATA) libpostgres.a '$(DESTDIR)$(libdir)/libpostgres.a'
endif
endif
ifeq ($(PORTNAME), win32)
ifeq ($(MAKE_DLL), true)
	$(INSTALL_DATA) libpostgres.a '$(DESTDIR)$(libdir)/libpostgres.a'
endif
endif
	$(MAKE) -C catalog install-data
	$(MAKE) -C tsearch install-data
	$(MAKE) -C utils install-data
	$(INSTALL_DATA) $(srcdir)/libpq/pg_hba.conf.sample '$(DESTDIR)$(datadir)/pg_hba.conf.sample'
	$(INSTALL_DATA) $(srcdir)/libpq/pg_ident.conf.sample '$(DESTDIR)$(datadir)/pg_ident.conf.sample'
	$(INSTALL_DATA) $(srcdir)/utils/misc/postgresql.conf.sample '$(DESTDIR)$(datadir)/postgresql.conf.sample'
	$(INSTALL_DATA) $(srcdir)/crypto/ckey_aws.sh.sample '$(DESTDIR)$(datadir)/auth_commands/ckey_aws.sh.sample'
	$(INSTALL_DATA) $(srcdir)/crypto/ckey_direct.sh.sample '$(DESTDIR)$(datadir)/auth_commands/ckey_direct.sh.sample'
	$(INSTALL_DATA) $(srcdir)/crypto/ckey_passphrase.sh.sample '$(DESTDIR)$(datadir)/auth_commands/ckey_passphrase.sh.sample'
	$(INSTALL_DATA) $(srcdir)/crypto/ckey_piv_nopin.sh.sample '$(DESTDIR)$(datadir)/auth_commands/ckey_piv_nopin.sh.sample'
	$(INSTALL_DATA) $(srcdir)/crypto/ckey_piv_pin.sh.sample '$(DESTDIR)$(datadir)/auth_commands/ckey_piv_pin.sh.sample'
	$(INSTALL_DATA) $(srcdir)/crypto/ssl_passphrase.sh.sample '$(DESTDIR)$(datadir)/auth_commands/ssl_passphrase.sh.sample'
ifeq ($(enable_orca), yes)
	$(MAKE) -C gporca $@ INSTLOC=$(DESTDIR)$(libdir)
	$(MAKE) -C gpopt $@ INSTLOC=$(DESTDIR)$(libdir)
endif
ifeq ($(enable_catalog_ext), yes)
	$(MAKE) -C catalog-extension $@ INSTLOC=$(DESTDIR)$(libdir)
endif

ifeq ($(with_llvm), yes)
install-bin: install-postgres-bitcode

# GPDB: Bitcode generation in certain subdir can be avoid when `with_llvm = no`
# set there. But the install step is not aware of the setting and would fail if
# we don't tell it the absence of *.bc files under those directory manually.
# As we set ORCA to generate no bitcodes, ORCA directories is excluded here.
bitcode_ignored_subdirs = $(top_builddir)/src/timezone gporca gpopt

install-postgres-bitcode: $(OBJS) all
	$(call install_llvm_module,postgres,$(call expand_subsys, $(filter-out $(bitcode_ignored_subdirs:%=%/objfiles.txt), $(SUBDIROBJS))))
endif

install-bin: postgres $(POSTGRES_IMP) installdirs
	$(INSTALL_PROGRAM) postgres$(X) '$(DESTDIR)$(bindir)/postgres$(X)'
ifeq ($(enable_shared_postgres_backend),yes)
	$(INSTALL_PROGRAM) libpostgres.so '$(DESTDIR)$(libdir)/libpostgres.so'
endif
ifneq ($(PORTNAME), win32)
	@rm -f $(DESTDIR)$(bindir)/postmaster$(X)
	ln -s postgres$(X) $(DESTDIR)$(bindir)/postmaster$(X)
else
	$(INSTALL_PROGRAM) postgres$(X) '$(DESTDIR)$(bindir)/postmaster$(X)'
endif
ifeq ($(MAKE_EXPORTS), true)
	$(INSTALL_DATA) $(POSTGRES_IMP) '$(DESTDIR)$(pkglibdir)/$(POSTGRES_IMP)'
	$(INSTALL_PROGRAM) $(MKLDEXPORT) '$(DESTDIR)$(pgxsdir)/$(MKLDEXPORT_DIR)/mkldexport.sh'
endif

.PHONY: install-bin

installdirs:
	$(MKDIR_P) '$(DESTDIR)$(bindir)' '$(DESTDIR)$(datadir)'
	$(MKDIR_P) '$(DESTDIR)$(datadir)' '$(DESTDIR)$(datadir)/auth_commands'
ifeq ($(PORTNAME), cygwin)
ifeq ($(MAKE_DLL), true)
	$(MKDIR_P) '$(DESTDIR)$(libdir)'
endif
endif
ifeq ($(PORTNAME), win32)
ifeq ($(MAKE_DLL), true)
	$(MKDIR_P) '$(DESTDIR)$(libdir)'
endif
endif
ifeq ($(MAKE_EXPORTS), true)
	$(MKDIR_P) '$(DESTDIR)$(pkglibdir)'
	$(MKDIR_P) '$(DESTDIR)$(pgxsdir)/$(MKLDEXPORT_DIR)'
endif


##########################################################################

uninstall:
	rm -f $(DESTDIR)$(bindir)/postgres$(X) $(DESTDIR)$(bindir)/postmaster $(DESTDIR)$(bindir)/cdbsyncmaster
ifeq ($(enable_shared_postgres_backend),yes)
	rm -f $(DESTDIR)$(bindir)/postgres$(X) $(DESTDIR)$(libdir)/libpostgres.so $(DESTDIR)$(bindir)/postmaster $(DESTDIR)$(bindir)/cdbsyncmaster
endif
ifeq ($(MAKE_EXPORTS), true)
	rm -f '$(DESTDIR)$(pkglibdir)/$(POSTGRES_IMP)'
	rm -f '$(DESTDIR)$(pgxsdir)/$(MKLDEXPORT_DIR)/mkldexport.sh'
endif
ifeq ($(PORTNAME), cygwin)
ifeq ($(MAKE_DLL), true)
	rm -f '$(DESTDIR)$(libdir)/libpostgres.a'
endif
endif
ifeq ($(PORTNAME), win32)
ifeq ($(MAKE_DLL), true)
	rm -f '$(DESTDIR)$(libdir)/libpostgres.a'
endif
endif
	$(MAKE) -C catalog uninstall-data
	$(MAKE) -C tsearch uninstall-data
	$(MAKE) -C utils uninstall-data
	rm -f '$(DESTDIR)$(datadir)/pg_hba.conf.sample' \
	      '$(DESTDIR)$(datadir)/pg_ident.conf.sample' \
	      '$(DESTDIR)$(datadir)/postgresql.conf.sample' \
	      '$(DESTDIR)$(datadir)/auth_commands/ckey_aws.sh.sample' \
	      '$(DESTDIR)$(datadir)/auth_commands/ckey_direct.sh.sample' \
	      '$(DESTDIR)$(datadir)/auth_commands/ckey_passphrase.sh.sample' \
	      '$(DESTDIR)$(datadir)/auth_commands/ckey_piv_nopin.sh.sample' \
	      '$(DESTDIR)$(datadir)/auth_commands/ckey_piv_pin.sh.sample' \
	      '$(DESTDIR)$(datadir)/auth_commands/ssl_passphrase.sh.sample'
ifeq ($(with_llvm), yes)
	$(call uninstall_llvm_module,postgres)
endif


##########################################################################

clean:
	rm -f $(LOCALOBJS) postgres$(X) $(POSTGRES_IMP)
ifeq ($(enable_shared_postgres_backend),yes)
	rm -f $(LOCALOBJS) libpostgres.so $(POSTGRES_IMP)
endif
ifeq ($(PORTNAME), cygwin)
	rm -f postgres.dll libpostgres.a
endif
ifeq ($(PORTNAME), win32)
	rm -f postgres.dll libpostgres.a $(WIN32RES)
endif
ifeq (,$(filter cygwin win32 aix,$(PORTNAME)))
	rm -f postgres_symbols.map
	rm -f hide_symbols.list
endif

distclean: clean
	rm -f port/tas.s port/pg_sema.c port/pg_shmem.c

maintainer-clean: distclean
	$(MAKE) -C catalog $@
	$(MAKE) -C utils $@
	rm -f bootstrap/bootparse.c \
	      bootstrap/bootscanner.c \
	      parser/gram.c \
	      parser/gram.h \
	      parser/scan.c \
	      replication/repl_gram.c \
	      replication/repl_scanner.c \
	      replication/syncrep_gram.c \
	      replication/syncrep_scanner.c \
	      storage/lmgr/lwlocknames.c \
	      storage/lmgr/lwlocknames.h \
	      utils/adt/jsonpath_gram.c \
	      utils/adt/jsonpath_scan.c \
	      utils/misc/guc-file.c


##########################################################################
#
# Support for code development.
#
# Use target "quick" to build "postgres" when you know all the subsystems
# are up to date.  It saves the time of doing all the submakes.
.PHONY: quick
quick: $(OBJS)
	$(CXX) $(CXXFLAGS) $(call expand_subsys,$^) $(LDFLAGS) $(LDFLAGS_EX) $(export_dynamic) $(LIBS) -o postgres
