cmake_minimum_required(VERSION 3.1.0 FATAL_ERROR)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake;${CMAKE_MODULE_PATH}")

include(CheckCXXSourceCompiles)
include(CheckIncludeFile)
include(CheckFunctionExists)
include(CheckStructHasMember)
include(CheckSymbolExists)
include(CheckTypeSize)

project(aria2 VERSION 1.34.0 LANGUAGES C CXX)

set(PACKAGE ${CMAKE_PROJECT_NAME})
set(PACKAGE_NAME ${CMAKE_PROJECT_NAME})
set(PACKAGE_VERSION ${CMAKE_PROJECT_VERSION})
set(PACKAGE_BUGREPORT "https://github.com/aria2/aria2/issues")
set(PACKAGE_URL "https://aria2.github.io/")
message("PACKAGE_NAME: ${PACKAGE_NAME}, PACKAGE_VERSION: ${PACKAGE_VERSION}")

set(BUILD "${CMAKE_HOST_SYSTEM_PROCESSOR}-${CMAKE_HOST_SYSTEM}")
set(TARGET "${CMAKE_SYSTEM_PROCESSOR}-${CMAKE_SYSTEM}")

option(ENABLE_SSL "Enable ssl" ON)
option(ENABLE_BITTORRENT "Enable bittorrent" ON)
option(ENABLE_METALINK "Enable metalink" ON)
option(ENABLE_WEBSOCKET "Enable websocket" ON)
option(ENABLE_EPOLL "Enable epoll" ON)
option(ENABLE_LIBARIA2 "Enable libaria2" OFF)
option(ENABLE_WERROR "Enable werror" OFF)

option(WITH_LIBNETTLE "Build with libnettle library" OFF)
option(WITH_LIBGMP "Build with libgmp library" OFF)
option(WITH_LIBGCRYPT "Build with libgcrypt library" OFF)

option(WITH_LIBXML2 "Build with libxml2 library" OFF)
option(WITH_LIBEXPAT "Build with libexpat library" OFF)

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    option(WITH_APPLETLS "Apple OSX the OS-level SSL/TLS support." ON)
elseif (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    option(WITH_WINTLS "Use SSL implementation based on the native Windows SSL capabilities" ON)
endif ()

set(LIBRARIES "")

# Setup SSL
if (ENABLE_SSL)
    if (WITH_APPLETLS)
        set(HAVE_APPLETLS TRUE)
        list(APPEND LIBRARIES "-framework CoreFoundation -framework Security")
    elseif (WITH_WINTLS)
        find_library(CRYPT32 crypt32)
        find_library(SECUR32 secur32)
        find_library(ADVAPI32 advapi32)
        check_include_file("wincrypt.h" HAVE_WINCRYPT_H)
        check_include_file("security.h" HAVE_SECURITY_H)
        if (CRYPT32 AND SECUR32 AND ADVAPI32 AND HAVE_WINCRYPT_H AND HAVE_SECURITY_H)
            set(HAVE_WINTLS TRUE)
            set(ENABLE_SSL ON)
            set(SECURITY_WIN32 1)
            list(APPEND LIBRARIES "-lcrypt32 -lsecur32 -ladvapi32")
        endif ()
    endif ()

    if (NOT HAVE_APPLETLS AND NOT HAVE_WINTLS)
        find_package(gnutls)
        if (GNUTLS_FOUND)
            set(HAVE_LIBGNUTLS TRUE)
            set(CMAKE_REQUIRED_LIBRARIES "${GNUTLS_LIBS}")
            set(CMAKE_REQUIRED_INCLUDES "${GNUTLS_LIBS}")
            check_function_exists(gnutls_certificate_set_x509_system_trust HAVE_GNUTLS_CERTIFICATE_SET_X509_SYSTEM_TRUST)
            unset(CMAKE_REQUIRED_LIBRARIES)
            if (HAVE_GNUTLS_CERTIFICATE_SET_X509_SYSTEM_TRUST)
                list(APPEND LIBRARIES LIBGNUTLS)
            endif ()
        endif (GNUTLS_FOUND)
    endif ()

    if (NOT HAVE_APPLETLS AND NOT HAVE_WINTLS AND NOT HAVE_LIBGNUTLS)
        find_package(openssl "0.9.8" REQUIRED)
        if (OPENSSL_FOUND)
            set(HAVE_OPENSSL TRUE)
            check_function_exists(EVP_DigestInit_ex HAVE_DIGESTINIT_EX)
            if (NOT HAVE_DIGESTINIT_EX)
                set(HAVE_OLD_OPENSSL 1)
            endif (NOT HAVE_DIGESTINIT_EX)
            check_function_exists(EVP_sha224 HAVE_EVP_SHA224)
            check_function_exists(EVP_sha256 HAVE_EVP_SHA256)
            check_function_exists(EVP_sha384 HAVE_EVP_SHA384)
            check_function_exists(EVP_sha512 HAVE_EVP_SHA512)
        endif ()
    endif ()
endif (ENABLE_SSL)


if (NOT HAVE_OPENSSL)
    if (WITH_LIBNETTLE AND NOT HAVE_NATIVETLS)
        find_package(nettle)
        if (NETTLE_FOUND)
            set(HAVE_LIBNETTLE TRUE)
        else (NETTLE_FOUND)
            message(FATAL_ERROR "WITH_LIBNETTLE specified but libnettle not found.")
        endif(NETTLE_FOUND)
    endif()
    if (WITH_LIBGMP)
        if ((HAVE_LIBNETTLE OR HAVE_NATIVETLS) AND ENABLE_BITTORRENT)
            find_library(gmp)
            if (gmp_FOUND)
                set(HAVE_LIBGMP TRUE)
                set(CMAKE_REQUIRED_LIBRARIES "${gmp}")
                check_function_exists(__gmpz_powm_sec HAVE_MPZ_POWM_SEC)
            else(GMP_FOUND)
                message(FATAL_ERROR "WITH_LIBGMP specified but libgmp not found.")
            endif(GMP_FOUND)
        endif()
    endif(WITH_LIBGMP)

    if (WITH_LIBGCRYPT)
        if (NOT HAVE_NATIVETLS AND NOT HAVE_LIBNETTLE)
            find_library(crypt)
            if (crypt)
                set(HAVE_LIBGCRYPT TRUE)
            endif(crypt)
        endif()
    endif(WITH_LIBGCRYPT)
endif()


# Set Message digist
if (HAVE_APPLETLS)
    set(USE_APPLE_MD TRUE)
elseif (HAVE_LIBNETTLE)
    set(USE_LIBNETTLE_MD TRUE)
elseif (HAVE_LIBGCRYPT)
    set(HAVE_LIBGCRYPT_MD TRUE)
elseif (HAVE_OPENSSL)
    set(USE_OPENSSL_MD TRUE)
else ()
    set(USE_INTERNAL_MD TRUE)
endif ()

set(USE_INTERNAL_ARC4 (NOT HAVE_LIBNETTLE AND NOT HAVE_LIBGCRYPT AND NOT HAVE_OPENSSL))
set(USE_INTERNAL_BIGNUM (NOT HAVE_LIBGMP AND NOT HAVE_LIBGCRYPT AND NOT HAVE_OPENSSL))

if (WITH_LIBXML2)
    find_package(LIBXML2)
    if (LIBXML2_FOUND)
        set(HAVE_LIBXML2 TRUE)
    else(LIBXML2_FOUND)
        message(FATAL_ERROR "WITH_LIBXML2 specified but libxml2 not found.")
    endif(LIBXML2_FOUND)
endif(WITH_LIBXML2)

if (WITH_LIBEXPAT)
    find_package(EXPAT)
    if (EXPAT_FOUND)
        set(HAVE_LIBEXPAT TRUE)
    else(EXPAT_FOUND)
        message(FATAL_ERROR "WITH_LIBEXPAT specified but libexpat not found.")
    endif(LIBXML2_FOUND)
endif(WITH_LIBEXPAT)

if (HAVE_LIBXML2 OR HAVE_LIBEXPAT)
    set(HAVE_SOME_XMLLIB TRUE)
    set(ENABLE_XML_RPC TRUE)
else()
    set(ENABLE_METALINK FALSE)
    set(ENABLE_XML_RPC FALSE)
endif(HAVE_LIBXML2 OR HAVE_LIBEXPAT)




check_struct_has_member("struct timespec" tv_sec time.h HAVE_A2_STRUCT_TIMESPEC)


check_include_file("argz.h" HAVE_ARGZ_H)
check_include_file("arpa/inet.h" HAVE_ARPA_INET_H)
check_include_file("fcntl.h" HAVE_FCNTL_H)
check_include_file("float.h" HAVE_FLOAT_H)
check_include_file("inttypes.h" HAVE_INTTYPES_H)
check_include_file("langinfo.h" HAVE_LANGINFO_H)
check_include_file("libintl.h" HAVE_LIBINTL_H)
check_include_file("limits.h" HAVE_LIMITS_H)
check_include_file("locale.h" HAVE_LOCALE_H)
check_include_file("malloc.h" HAVE_MALLOC_H)
check_include_file("netdb.h" HAVE_NETDB_H)
check_include_file("netinet/in.h" HAVE_NETINET_IN_H)
check_include_file("netinet/tcp.h" HAVE_NETINET_TCP_H)
check_include_file("poll.h" HAVE_POLL_H)
check_include_file("port.h" HAVE_PORT_H)
check_include_file("signal.h" HAVE_SIGNAL_H)
check_include_file("stddef.h" HAVE_STDDEF_H)
check_include_file("stdint.h" HAVE_STDINT_H)
check_include_file("stdio_ext.h" HAVE_STDIO_EXT_H)
check_include_file("stdlib.h" HAVE_STDLIB_H)
check_include_file("string.h" HAVE_STRING_H)
check_include_file("strings.h" HAVE_STRINGS_H)
check_include_file("sys/ioctl.h" HAVE_SYS_IOCTL_H)
check_include_file("sys/param.h" HAVE_SYS_PARAM_H)
check_include_file("sys/resource.h" HAVE_SYS_RESOURCE_H)
check_include_file("sys/signal.h" HAVE_SYS_SIGNAL_H)
check_include_file("sys/socket.h" HAVE_SYS_SOCKET_H)
check_include_file("sys/time.h" HAVE_SYS_TIME_H)
check_include_file("sys/types.h" HAVE_SYS_TYPES_H)
check_include_file("sys/uio.h" HAVE_SYS_UIO_H)
check_include_file("sys/utsname.h" HAVE_SYS_UTSNAME_H)
check_include_file("termios.h" HAVE_TERMIOS_H)
check_include_file("unistd.h" HAVE_UNISTD_H)
check_include_file("utime.h" HAVE_UTIME_H)
check_include_file("wchar.h" HAVE_WCHAR_H)
check_include_file("ifaddrs.h" HAVE_IFADDRS_H)
check_include_file("pwd.h" HAVE_PWD_H)

check_function_exists(__argz_count HAVE___ARGZ_COUNT)
check_function_exists(__argz_next HAVE___ARGZ_NEXT)
check_function_exists(__argz_stringify HAVE___ARGZ_STRINGIFY)
check_function_exists(atexit HAVE_ATEXIT)
check_function_exists(fork HAVE_FORK)
check_function_exists(ftruncate HAVE_FTRUNCATE)
check_function_exists(getcwd HAVE_GETCWD)
check_function_exists(gethostbyaddr HAVE_GETHOSTBYADDR)
check_function_exists(gethostbyname HAVE_GETHOSTBYNAME)
check_function_exists(getifaddrs HAVE_GETIFADDRS)
check_function_exists(getpagesize HAVE_GETPAGESIZE)
check_function_exists(memchr HAVE_MEMCHR)
check_function_exists(memcmp HAVE_MEMCMP)
check_function_exists(memmove HAVE_MEMMOVE)
check_function_exists(mempcpy HAVE_MEMPCPY)
check_function_exists(memset HAVE_MEMSET)
check_function_exists(mkdir HAVE_MKDIR)
check_function_exists(mktime HAVE_MKTIME)
check_function_exists(mmap HAVE_MMAP)
check_function_exists(munmap HAVE_MUNMAP)
check_function_exists(nl_langinfo HAVE_NL_LANGINFO)
check_function_exists(posix_fadvise HAVE_POSIX_FADVISE)
check_function_exists(posix_memalign HAVE_POSIX_MEMALIGN)
check_function_exists(pow HAVE_POW)
check_function_exists(putenv HAVE_PUTENV)
check_function_exists(rmdir HAVE_RMDIR)
check_function_exists(select HAVE_SELECT)
check_function_exists(setlocale HAVE_SETLOCALE)
check_function_exists(sigaction HAVE_SIGACTION)
check_function_exists(sleep HAVE_SLEEP)
check_function_exists(socket HAVE_SOCKET)
check_function_exists(stat HAVE_STAT)
check_function_exists(stpcpy HAVE_STPCPY)
check_function_exists(strcasecmp HAVE_STRCASECMP)
check_function_exists(strchr HAVE_STRCHR)
check_function_exists(strcspn HAVE_STRCSPN)
check_function_exists(strdup HAVE_STRDUP)
check_function_exists(strerror HAVE_STRERROR)
check_function_exists(strftime HAVE_STRFTIME)
check_function_exists(strncasecmp HAVE_STRNCASECMP)
check_function_exists(strstr HAVE_STRSTR)
check_function_exists(strtod HAVE_STRTOD)
check_function_exists(strtol HAVE_STRTOL)
check_function_exists(strtoul HAVE_STRTOUL)
check_function_exists(strtoull HAVE_STRTOULL)
check_function_exists(tzset HAVE_TZSET)
check_function_exists(unsetenv HAVE_UNSETENV)
check_function_exists(usleep HAVE_USLEEP)
check_function_exists(utime HAVE_UTIME)
check_function_exists(utimes HAVE_UTIMES)
check_function_exists(vprintf HAVE_VPRINTF)

check_function_exists(basename HAVE_BASENAME)
check_function_exists(gai_strerror HAVE_GAI_STRERROR)
check_function_exists(getaddrinfo HAVE_GETADDRINFO)
check_function_exists(gettimeofday HAVE_GETTIMEOFDAY)
check_function_exists(strptime HAVE_STRPTIME)
check_function_exists(poll HAVE_POLL)

check_type_size("size_t" SIZEOF_SIZE_T)
if (NOT SIZEOF_SIZE_T)
    set(size_t "unsigned int")
endif ()

check_cxx_source_compiles(
    "
        struct Base {virtual void f() = 0;};
        struct Derived : Base { virtual void f() override {}};
        int main() {return 0;}
    "
    HAVE_CXX11_OVERRIDE
)
if (HAVE_CXX11_OVERRIDE)
    set(CXX11_OVERRIDE "override")
endif ()


find_package(LibSSH2)
if (LIBSSH2_FOUND)
    set(HAVE_LIBSSH2 TRUE)
    include_directories(${LIBSSH2_INCLUDE_DIR})
    list(APPEND LIBRARIES ${LIBSSH2_LIBRARY})
endif ()


set(CMAKE_INCLUDE_CURRENT_DIR ON)
configure_file(config.h.cmake.in config.h)
add_definitions(-DHAVE_CONFIG_H)
include_directories(src src/includes)


set(SRCS
    src/a2algo.h
    src/a2functional.h
    src/a2io.h
    src/a2iterator.h
    src/a2netcompat.h
    src/A2STR.cc src/A2STR.h
    src/a2time.h
    src/AbstractAuthResolver.cc src/AbstractAuthResolver.h
    src/AbstractCommand.cc src/AbstractCommand.h
    src/AbstractDiskWriter.cc src/AbstractDiskWriter.h
    src/AbstractHttpServerResponseCommand.cc
    src/AbstractHttpServerResponseCommand.h
    src/AbstractOptionHandler.cc src/AbstractOptionHandler.h
    src/AbstractProxyRequestCommand.cc src/AbstractProxyRequestCommand.h
    src/AbstractProxyResponseCommand.cc src/AbstractProxyResponseCommand.h
    src/AbstractSingleDiskAdaptor.cc src/AbstractSingleDiskAdaptor.h
    src/AdaptiveFileAllocationIterator.cc src/AdaptiveFileAllocationIterator.h
    src/AdaptiveURISelector.cc src/AdaptiveURISelector.h
    src/AnonDiskWriterFactory.h
    src/array_fun.h
    src/AuthConfig.cc src/AuthConfig.h
    src/AuthConfigFactory.cc src/AuthConfigFactory.h
    src/AuthResolver.h
    src/AutoSaveCommand.cc src/AutoSaveCommand.h
    src/BackupIPv4ConnectCommand.cc src/BackupIPv4ConnectCommand.h
    src/base32.cc src/base32.h
    src/base64.h
    src/BinaryStream.h
    src/bitfield.cc src/bitfield.h
    src/BitfieldMan.cc src/BitfieldMan.h
    src/BtProgressInfoFile.h
    src/BufferedFile.cc src/BufferedFile.h
    src/ByteArrayDiskWriter.cc src/ByteArrayDiskWriter.h
    src/ByteArrayDiskWriterFactory.h
    src/CheckIntegrityCommand.cc src/CheckIntegrityCommand.h
    src/CheckIntegrityDispatcherCommand.cc src/CheckIntegrityDispatcherCommand.h
    src/CheckIntegrityEntry.cc src/CheckIntegrityEntry.h
    src/CheckIntegrityMan.h
    src/Checksum.cc src/Checksum.h
    src/ChecksumCheckIntegrityEntry.cc src/ChecksumCheckIntegrityEntry.h
    src/ChunkChecksum.cc src/ChunkChecksum.h
    src/ChunkedDecodingStreamFilter.cc src/ChunkedDecodingStreamFilter.h
    src/ColorizedStream.cc src/ColorizedStream.h
    src/Command.cc src/Command.h
    src/common.h
    src/ConnectCommand.cc src/ConnectCommand.h
    src/console.cc src/console.h
    src/ConsoleStatCalc.cc src/ConsoleStatCalc.h
    src/ContentTypeRequestGroupCriteria.cc src/ContentTypeRequestGroupCriteria.h
    src/Context.cc src/Context.h
    src/ContextAttribute.cc src/ContextAttribute.h
    src/ControlChain.h
    src/Cookie.cc src/Cookie.h
    src/CookieStorage.cc src/CookieStorage.h
    src/cookie_helper.cc src/cookie_helper.h
    src/CreateRequestCommand.cc src/CreateRequestCommand.h
    src/crypto_endian.h
    src/CUIDCounter.cc src/CUIDCounter.h
    src/DefaultAuthResolver.cc src/DefaultAuthResolver.h
    src/DefaultBtProgressInfoFile.cc src/DefaultBtProgressInfoFile.h
    src/DefaultDiskWriter.cc src/DefaultDiskWriter.h
    src/DefaultDiskWriterFactory.cc src/DefaultDiskWriterFactory.h
    src/DefaultPieceStorage.cc src/DefaultPieceStorage.h
    src/DefaultStreamPieceSelector.cc src/DefaultStreamPieceSelector.h
    src/DelayedCommand.h
    src/Dependency.h
    src/DirectDiskAdaptor.cc src/DirectDiskAdaptor.h
    src/DiskAdaptor.cc src/DiskAdaptor.h
    src/DiskWriter.h
    src/DiskWriterFactory.h
    src/DlAbortEx.cc src/DlAbortEx.h
    src/DlRetryEx.cc src/DlRetryEx.h
    src/DNSCache.cc src/DNSCache.h
    src/DownloadCommand.cc src/DownloadCommand.h
    src/DownloadContext.cc src/DownloadContext.h
    src/DownloadEngine.cc src/DownloadEngine.h
    src/DownloadEngineFactory.cc src/DownloadEngineFactory.h
    src/DownloadFailureException.cc src/DownloadFailureException.h
    src/DownloadHandler.cc src/DownloadHandler.h
    src/DownloadHandlerConstants.cc src/DownloadHandlerConstants.h
    src/DownloadResult.cc src/DownloadResult.h
    src/download_handlers.cc src/download_handlers.h
    src/download_helper.cc src/download_helper.h
    src/error_code.h
    src/Event.h
    src/EventPoll.h
    src/Exception.cc src/Exception.h
    src/FatalException.cc src/FatalException.h
    src/FeatureConfig.cc src/FeatureConfig.h
    src/FeedbackURISelector.cc src/FeedbackURISelector.h
    src/File.cc src/File.h
    src/FileAllocationCommand.cc src/FileAllocationCommand.h
    src/FileAllocationDispatcherCommand.cc src/FileAllocationDispatcherCommand.h
    src/FileAllocationEntry.cc src/FileAllocationEntry.h
    src/FileAllocationIterator.h
    src/FileAllocationMan.h
    src/FileEntry.cc src/FileEntry.h
    src/FillRequestGroupCommand.cc src/FillRequestGroupCommand.h
    src/fmt.cc src/fmt.h
    src/FtpConnection.cc src/FtpConnection.h
    src/FtpDownloadCommand.cc src/FtpDownloadCommand.h
    src/FtpFinishDownloadCommand.cc src/FtpFinishDownloadCommand.h
    src/FtpInitiateConnectionCommand.cc src/FtpInitiateConnectionCommand.h
    src/FtpNegotiationCommand.cc src/FtpNegotiationCommand.h
    src/FtpNegotiationConnectChain.h
    src/FtpTunnelRequestCommand.cc src/FtpTunnelRequestCommand.h
    src/FtpTunnelRequestConnectChain.h
    src/FtpTunnelResponseCommand.cc src/FtpTunnelResponseCommand.h
    src/GenericParser.h
    src/GeomStreamPieceSelector.cc src/GeomStreamPieceSelector.h
    src/GroupId.cc src/GroupId.h
    src/GrowSegment.cc src/GrowSegment.h
    src/HashFuncEntry.h
    src/HaveEraseCommand.cc src/HaveEraseCommand.h
    src/help_tags.cc src/help_tags.h
    src/HttpConnection.cc src/HttpConnection.h
    src/HttpDownloadCommand.cc src/HttpDownloadCommand.h
    src/HttpHeader.cc src/HttpHeader.h
    src/HttpHeaderProcessor.cc src/HttpHeaderProcessor.h
    src/HttpInitiateConnectionCommand.cc src/HttpInitiateConnectionCommand.h
    src/HttpListenCommand.cc src/HttpListenCommand.h
    src/HttpProxyRequestCommand.cc src/HttpProxyRequestCommand.h
    src/HttpProxyRequestConnectChain.h
    src/HttpProxyResponseCommand.cc src/HttpProxyResponseCommand.h
    src/HttpRequest.cc src/HttpRequest.h
    src/HttpRequestCommand.cc src/HttpRequestCommand.h
    src/HttpRequestConnectChain.h
    src/HttpResponse.cc src/HttpResponse.h
    src/HttpResponseCommand.cc src/HttpResponseCommand.h
    src/HttpServer.cc src/HttpServer.h
    src/HttpServerBodyCommand.cc src/HttpServerBodyCommand.h
    src/HttpServerCommand.cc src/HttpServerCommand.h
    src/HttpServerResponseCommand.cc src/HttpServerResponseCommand.h
    src/HttpSkipResponseCommand.cc src/HttpSkipResponseCommand.h
    src/IndexedList.h
    src/InitiateConnectionCommand.cc src/InitiateConnectionCommand.h
    src/InitiateConnectionCommandFactory.cc src/InitiateConnectionCommandFactory.h
    src/InorderStreamPieceSelector.cc src/InorderStreamPieceSelector.h
    src/RandomStreamPieceSelector.cc src/RandomStreamPieceSelector.h
    src/InorderURISelector.cc src/InorderURISelector.h
    src/IOFile.cc src/IOFile.h
    src/IteratableChecksumValidator.cc src/IteratableChecksumValidator.h
    src/IteratableChunkChecksumValidator.cc src/IteratableChunkChecksumValidator.h
    src/IteratableValidator.h
    src/json.cc src/json.h
    src/JsonDiskWriter.h
    src/JsonParser.cc src/JsonParser.h
    src/Lock.h
    src/LogFactory.cc src/LogFactory.h
    src/Logger.cc src/Logger.h
    src/LongestSequencePieceSelector.cc src/LongestSequencePieceSelector.h
    src/MemoryBufferPreDownloadHandler.h
    src/MemoryPreDownloadHandler.h
    src/message.h
    src/MessageDigest.cc src/MessageDigest.h
    src/MessageDigestImpl.h
    src/message_digest_helper.cc src/message_digest_helper.h
    src/MetadataInfo.cc src/MetadataInfo.h
    src/MetalinkHttpEntry.cc src/MetalinkHttpEntry.h
    src/MultiDiskAdaptor.cc src/MultiDiskAdaptor.h
    src/MultiFileAllocationIterator.cc src/MultiFileAllocationIterator.h
    src/MultiUrlRequestInfo.cc src/MultiUrlRequestInfo.h
    src/NameResolver.cc src/NameResolver.h
    src/Netrc.cc src/Netrc.h
    src/NetrcAuthResolver.cc src/NetrcAuthResolver.h
    src/NetStat.cc src/NetStat.h
    src/Notifier.cc src/Notifier.h
    src/NsCookieParser.cc src/NsCookieParser.h
    src/NullHandle.h
    src/NullOutputFile.h
    src/NullProgressInfoFile.h
    src/NullSinkStreamFilter.cc src/NullSinkStreamFilter.h
    src/NullStatCalc.h
    src/Option.cc src/Option.h
    src/OptionHandler.cc src/OptionHandler.h
    src/OptionHandlerException.cc src/OptionHandlerException.h
    src/OptionHandlerFactory.cc src/OptionHandlerFactory.h
    src/OptionHandlerImpl.cc src/OptionHandlerImpl.h
    src/OptionParser.cc src/OptionParser.h
    src/option_processing.cc
    src/OutputFile.h
    src/paramed_string.cc src/paramed_string.h
    src/PeerStat.cc src/PeerStat.h
    src/Piece.cc src/Piece.h
    src/PiecedSegment.cc src/PiecedSegment.h
    src/PieceHashCheckIntegrityEntry.cc src/PieceHashCheckIntegrityEntry.h
    src/PieceSelector.h
    src/PieceStatMan.cc src/PieceStatMan.h
    src/PieceStorage.h
    src/Platform.cc src/Platform.h
    src/PostDownloadHandler.h
    src/PreDownloadHandler.h
    src/prefs.cc src/prefs.h
    src/ProgressAwareEntry.h
    src/ProtocolDetector.cc src/ProtocolDetector.h
    src/Randomizer.h
    src/Range.cc src/Range.h
    src/RarestPieceSelector.cc src/RarestPieceSelector.h
    src/RealtimeCommand.cc src/RealtimeCommand.h
    src/RecoverableException.cc src/RecoverableException.h
    src/Request.cc src/Request.h
    src/RequestGroup.cc src/RequestGroup.h
    src/RequestGroupCriteria.h
    src/RequestGroupEntry.cc src/RequestGroupEntry.h
    src/RequestGroupMan.cc src/RequestGroupMan.h
    src/RpcMethod.cc src/RpcMethod.h
    src/RpcMethodFactory.cc src/RpcMethodFactory.h
    src/RpcMethodImpl.cc src/RpcMethodImpl.h
    src/RpcRequest.cc src/RpcRequest.h
    src/RpcResponse.cc src/RpcResponse.h
    src/rpc_helper.cc src/rpc_helper.h
    src/SaveSessionCommand.cc src/SaveSessionCommand.h
    src/SegList.h
    src/Segment.h
    src/SegmentMan.cc src/SegmentMan.h
    src/SelectEventPoll.cc src/SelectEventPoll.h
    src/SequentialDispatcherCommand.h
    src/SequentialPicker.h
    src/ServerStat.cc src/ServerStat.h
    src/ServerStatMan.cc src/ServerStatMan.h
    src/SessionSerializer.cc src/SessionSerializer.h
    src/Signature.cc src/Signature.h
    src/SimpleRandomizer.cc src/SimpleRandomizer.h
    src/SingleFileAllocationIterator.cc src/SingleFileAllocationIterator.h
    src/SingletonHolder.h
    src/SinkStreamFilter.cc src/SinkStreamFilter.h
    src/SocketBuffer.cc src/SocketBuffer.h
    src/SocketCore.cc src/SocketCore.h
    src/SocketRecvBuffer.cc src/SocketRecvBuffer.h
    src/SpeedCalc.cc src/SpeedCalc.h
    src/StatCalc.h
    src/StreamCheckIntegrityEntry.cc src/StreamCheckIntegrityEntry.h
    src/StreamFileAllocationEntry.cc src/StreamFileAllocationEntry.h
    src/StreamFilter.cc src/StreamFilter.h
    src/StreamPieceSelector.h
    src/StructParserStateMachine.h
    src/TimeA2.cc src/TimeA2.h
    src/TimeBasedCommand.cc src/TimeBasedCommand.h
    src/TimedHaltCommand.cc src/TimedHaltCommand.h
    src/TimerA2.cc src/TimerA2.h
    src/timespec.h
    src/TorrentAttribute.cc src/TorrentAttribute.h
    src/TransferStat.cc src/TransferStat.h
    src/TruncFileAllocationIterator.cc src/TruncFileAllocationIterator.h
    src/UnknownLengthPieceStorage.cc src/UnknownLengthPieceStorage.h
    src/UnknownOptionException.cc src/UnknownOptionException.h
    src/uri.cc src/uri.h
    src/UriListParser.cc src/UriListParser.h
    src/URIResult.cc src/URIResult.h
    src/URISelector.h
    src/uri_split.c src/uri_split.h
    src/usage_text.h
    src/util.cc src/util.h
    src/util_security.cc src/util_security.h
    src/ValueBase.cc src/ValueBase.h
    src/ValueBaseDiskWriter.h
    src/ValueBaseJsonParser.h
    src/ValueBaseStructParserState.h
    src/ValueBaseStructParserStateImpl.cc src/ValueBaseStructParserStateImpl.h
    src/ValueBaseStructParserStateMachine.cc src/ValueBaseStructParserStateMachine.h
    src/version_usage.cc
    src/wallclock.cc src/wallclock.h
    src/WatchProcessCommand.cc src/WatchProcessCommand.h
    src/WrDiskCache.cc src/WrDiskCache.h
    src/WrDiskCacheEntry.cc src/WrDiskCacheEntry.h
    src/XmlRpcRequestParserController.cc src/XmlRpcRequestParserController.h
    src/OpenedFileCounter.cc src/OpenedFileCounter.h
    src/SHA1IOFile.cc src/SHA1IOFile.h
    src/EvictSocketPoolCommand.cc src/EvictSocketPoolCommand.h
    src/libssl_compat.h
    )

if (MINGW)
    list(APPEND SRCS src/WinConsoleFile.cc src/WinConsoleFile.h)
endif ()

if (ENABLE_WEBSOCKET)
    list(APPEND SRCS
        src/WebSocketInteractionCommand.cc src/WebSocketInteractionCommand.h
        src/WebSocketResponseCommand.cc src/WebSocketResponseCommand.h
        src/WebSocketSession.cc src/WebSocketSession.h
        src/WebSocketSessionMan.cc src/WebSocketSessionMan.h
        )

        list(APPEND SRCS
            deps/wslay/lib/wslay_frame.c
            deps/wslay/lib/wslay_event.c
            deps/wslay/lib/wslay_queue.c
            deps/wslay/lib/wslay_net.c
            )
    include_directories(deps/wslay/lib/includes)
else ()
    list(APPEND SRCS src/NullWebSocketSessionMan.h)
endif ()

message("HAVE_SOME_XMLLIB: ${HAVE_SOME_XMLLIB}")
if (HAVE_SOME_XMLLIB)
    list(APPEND SRCS
        src/ParserStateMachine.h
        src/XmlAttr.cc src/XmlAttr.h
        src/XmlParser.cc src/XmlParser.h
        )
endif (HAVE_SOME_XMLLIB)

if (HAVE_LIBXML2)
    list(APPEND SRCS src/Xml2XmlParser.cc src/Xml2XmlParser.h)
endif (HAVE_LIBXML2)

if (HAVE_LIBEXPAT)
    list(APPEND SRCS src/xpatXmlParser.cc src/ExpatXmlParser.h)
endif (HAVE_LIBEXPAT)

if (ENABLE_XML_RPC)
    list(APPEND SRCS
        src/XmlRpcDiskWriter.cc src/XmlRpcDiskWriter.h
        src/XmlRpcRequestParserState.h
        src/XmlRpcRequestParserStateImpl.cc src/XmlRpcRequestParserStateImpl.h
        src/XmlRpcRequestParserStateMachine.cc src/XmlRpcRequestParserStateMachine.h
        )
endif (ENABLE_XML_RPC)

if (HAVE_SOME_FALLOCATE)
    list(APPEND SRCS src/FallocFileAllocationIterator.cc src/FallocFileAllocationIterator.h)
endif (HAVE_SOME_FALLOCATE)

if (HAVE_EPOLL)
    list(APPEND SRCS src/EpollEventPoll.cc src/EpollEventPoll.h)
endif (HAVE_EPOLL)

if (ENABLE_SSL)
    list(APPEND SRCS src/TLSContext.h src/TLSSession.h)
endif (ENABLE_SSL)

if (USE_APPLE_MD)
    list(APPEND SRCS src/AppleMessageDigestImpl.cc)
endif (USE_APPLE_MD)

if (HAVE_APPLETLS)
    list(APPEND SRCS
        src/AppleTLSContext.cc src/AppleTLSContext.h
        src/AppleTLSSession.cc src/AppleTLSSession.h
        )
endif (HAVE_APPLETLS)

if (HAVE_WINTLS)
    list(APPEND SRCS
    src/WinTLSContext.cc src/WinTLSContext.h
    src/WinTLSSession.cc src/WinTLSSession.h
        )
endif (HAVE_WINTLS)

if (USE_INTERNAL_BIGNUM)
    list(APPEND SRCS
    src/bignum.h
    src/InternalDHKeyExchange.cc src/InternalDHKeyExchange.h
        )
endif (USE_INTERNAL_BIGNUM)

if (USE_INTERNAL_ARC4)
    list(APPEND SRCS src/InternalARC4Encryptor.cc src/InternalARC4Encryptor.h)
endif (USE_INTERNAL_ARC4)

if (USE_INTERNAL_MD)
    list(APPEND SRCS
    src/InternalMessageDigestImpl.cc
    src/crypto_hash.cc src/crypto_hash.h
        )
endif (USE_INTERNAL_MD)

if (HAVE_LIBGNUTLS)
    list(APPEND SRCS
    src/LibgnutlsTLSContext.cc src/LibgnutlsTLSContext.h
    src/LibgnutlsTLSSession.cc src/LibgnutlsTLSSession.h
        )
endif (HAVE_LIBGNUTLS)

if (HAVE_LIBGCRYPT)
    list(APPEND SRCS
    src/LibgcryptARC4Encryptor.cc src/LibgcryptARC4Encryptor.h
    src/LibgcryptDHKeyExchange.cc src/LibgcryptDHKeyExchange.h
        )
    if (USE_LIBGCRYPT_MD)
        list(APPEND SRCS src/LibgcryptMessageDigestImpl.cc)
    endif (USE_LIBGCRYPT_MD)
endif (HAVE_LIBGCRYPT)

if (HAVE_LIBNETTLE)
    list(APPEND SRCS src/LibnettleARC4Encryptor.cc src/LibnettleARC4Encryptor.h)
    if (USE_LIBNETTLE_MD)
        list(APPEND SRCS src/LibnettleMessageDigestImpl.cc)
    endif (USE_LIBNETTLE_MD)
endif (HAVE_LIBNETTLE)

if (HAVE_LIBGMP)
    list(APPEND SRCS
    src/a2gmp.cc src/a2gmp.h
        src/LibgmpDHKeyExchange.cc src/LibgmpDHKeyExchange.h
        )
endif (HAVE_LIBGMP)

if (HAVE_OPENSSL)
    list(APPEND SRCS
    src/LibsslARC4Encryptor.cc src/LibsslARC4Encryptor.h
    src/LibsslDHKeyExchange.cc src/LibsslDHKeyExchange.h)
    if (NOT HAVE_APPLETLS)
        list(APPEND SRCS
        src/LibsslTLSContext.cc src/LibsslTLSContext.h
        src/LibsslTLSSession.cc src/LibsslTLSSession.h)
    endif (NOT HAVE_APPLETLS)
    if (USE_OPENSSL_MD)
        list(APPEND SRCS src/LibsslMessageDigestImpl.cc)
    endif (USE_OPENSSL_MD)
endif (HAVE_OPENSSL)

if (HAVE_ZLIB)
    list(APPEND SRCS
    src/GZipDecodingStreamFilter.cc src/GZipDecodingStreamFilter.h
    src/GZipEncoder.cc src/GZipEncoder.h
    src/GZipFile.cc src/GZipFile.h
    src/Adler32MessageDigestImpl.cc src/Adler32MessageDigestImpl.h)
endif (HAVE_ZLIB)

if (HAVE_SQLITE3)
    list(APPEND SRCS
    src/Sqlite3CookieParser.cc src/Sqlite3CookieParser.h
    src/Sqlite3CookieParserImpl.cc src/Sqlite3CookieParserImpl.h)
endif (HAVE_SQLITE3)

if (HAVE_LIBSSH2)
    list(APPEND SRCS
    src/SSHSession.cc src/SSHSession.h
    src/SftpNegotiationCommand.cc src/SftpNegotiationCommand.h
    src/SftpNegotiationConnectChain.h
    src/SftpDownloadCommand.cc src/SftpDownloadCommand.h
    src/SftpFinishDownloadCommand.cc src/SftpFinishDownloadCommand.h)
endif (HAVE_LIBSSH2)

if (ENABLE_ASYNC_DNS)
    list(APPEND SRCS
    src/AsyncNameResolver.cc src/AsyncNameResolver.h
    src/AsyncNameResolverMan.cc src/AsyncNameResolverMan.h)
endif (ENABLE_ASYNC_DNS)

if (ENABLE_BITTORRENT)
    list(APPEND SRCS
        src/AbstractBtMessage.cc src/AbstractBtMessage.h
        src/ActivePeerConnectionCommand.cc src/ActivePeerConnectionCommand.h
        src/AnnounceList.cc src/AnnounceList.h
        src/AnnounceTier.cc src/AnnounceTier.h
        src/ARC4Encryptor.h
        src/bencode2.cc src/bencode2.h
        src/BencodeDiskWriter.h
        src/BencodeDiskWriterFactory.h
        src/BencodeParser.cc src/BencodeParser.h
        src/bittorrent_helper.cc src/bittorrent_helper.h
        src/BtAbortOutstandingRequestEvent.cc src/BtAbortOutstandingRequestEvent.h
        src/BtAllowedFastMessage.cc src/BtAllowedFastMessage.h
        src/BtAnnounce.cc src/BtAnnounce.h
        src/BtBitfieldMessage.cc src/BtBitfieldMessage.h
        src/BtBitfieldMessageValidator.cc src/BtBitfieldMessageValidator.h
        src/BtCancelMessage.cc src/BtCancelMessage.h
        src/BtCancelSendingPieceEvent.h
        src/BtCheckIntegrityEntry.cc src/BtCheckIntegrityEntry.h
        src/BtChokeMessage.cc src/BtChokeMessage.h
        src/BtChokingEvent.h
        src/BtConstants.h
        src/BtDependency.cc src/BtDependency.h
        src/BtExtendedMessage.cc src/BtExtendedMessage.h
        src/BtFileAllocationEntry.cc src/BtFileAllocationEntry.h
        src/BtHandshakeMessage.cc src/BtHandshakeMessage.h
        src/BtHandshakeMessageValidator.cc src/BtHandshakeMessageValidator.h
        src/BtHaveAllMessage.cc src/BtHaveAllMessage.h
        src/BtHaveMessage.cc src/BtHaveMessage.h
        src/BtHaveNoneMessage.cc src/BtHaveNoneMessage.h
        src/BtInteractive.h
        src/BtInterestedMessage.cc src/BtInterestedMessage.h
        src/BtKeepAliveMessage.cc src/BtKeepAliveMessage.h
        src/BtLeecherStateChoke.cc src/BtLeecherStateChoke.h
        src/BtMessage.h
        src/BtMessageDispatcher.h
        src/BtMessageFactory.h
        src/BtMessageReceiver.h
        src/BtMessageValidator.h
        src/BtNotInterestedMessage.cc src/BtNotInterestedMessage.h
        src/BtPieceMessage.cc src/BtPieceMessage.h
        src/BtPieceMessageValidator.cc src/BtPieceMessageValidator.h
        src/BtPortMessage.cc src/BtPortMessage.h
        src/BtPostDownloadHandler.cc src/BtPostDownloadHandler.h
        src/BtRegistry.cc src/BtRegistry.h
        src/BtRejectMessage.cc src/BtRejectMessage.h
        src/BtRequestFactory.h
        src/BtRequestMessage.cc src/BtRequestMessage.h
        src/BtRuntime.cc src/BtRuntime.h
        src/BtSeederStateChoke.cc src/BtSeederStateChoke.h
        src/BtSetup.cc src/BtSetup.h
        src/BtStopDownloadCommand.cc src/BtStopDownloadCommand.h
        src/BtSuggestPieceMessage.cc src/BtSuggestPieceMessage.h
        src/BtUnchokeMessage.cc src/BtUnchokeMessage.h
        src/DefaultBtAnnounce.cc src/DefaultBtAnnounce.h
        src/DefaultBtInteractive.cc src/DefaultBtInteractive.h
        src/DefaultBtMessageDispatcher.cc src/DefaultBtMessageDispatcher.h
        src/DefaultBtMessageFactory.cc src/DefaultBtMessageFactory.h
        src/DefaultBtMessageReceiver.cc src/DefaultBtMessageReceiver.h
        src/DefaultBtRequestFactory.cc src/DefaultBtRequestFactory.h
        src/DefaultExtensionMessageFactory.cc src/DefaultExtensionMessageFactory.h
        src/DefaultPeerStorage.cc src/DefaultPeerStorage.h
        src/DHKeyExchange.h
        src/DHTAbstractMessage.cc src/DHTAbstractMessage.h
        src/DHTAbstractNodeLookupTask.h
        src/DHTAbstractTask.cc src/DHTAbstractTask.h
        src/DHTAnnouncePeerMessage.cc src/DHTAnnouncePeerMessage.h
        src/DHTAnnouncePeerReplyMessage.cc src/DHTAnnouncePeerReplyMessage.h
        src/DHTAutoSaveCommand.cc src/DHTAutoSaveCommand.h
        src/DHTBucket.cc src/DHTBucket.h
        src/DHTBucketRefreshCommand.cc src/DHTBucketRefreshCommand.h
        src/DHTBucketRefreshTask.cc src/DHTBucketRefreshTask.h
        src/DHTBucketTree.cc src/DHTBucketTree.h
        src/DHTConnection.h
        src/DHTConnectionImpl.cc src/DHTConnectionImpl.h
        src/DHTConstants.h
        src/DHTEntryPointNameResolveCommand.cc src/DHTEntryPointNameResolveCommand.h
        src/DHTFindNodeMessage.cc src/DHTFindNodeMessage.h
        src/DHTFindNodeReplyMessage.cc src/DHTFindNodeReplyMessage.h
        src/DHTGetPeersCommand.cc src/DHTGetPeersCommand.h
        src/DHTGetPeersMessage.cc src/DHTGetPeersMessage.h
        src/DHTGetPeersReplyMessage.cc src/DHTGetPeersReplyMessage.h
        src/DHTIDCloser.h
        src/DHTInteractionCommand.cc src/DHTInteractionCommand.h
        src/DHTMessage.cc src/DHTMessage.h
        src/DHTMessageCallback.h
        src/DHTMessageDispatcher.h
        src/DHTMessageDispatcherImpl.cc src/DHTMessageDispatcherImpl.h
        src/DHTMessageEntry.cc src/DHTMessageEntry.h
        src/DHTMessageFactory.h
        src/DHTMessageFactoryImpl.cc src/DHTMessageFactoryImpl.h
        src/DHTMessageReceiver.cc src/DHTMessageReceiver.h
        src/DHTMessageTracker.cc src/DHTMessageTracker.h
        src/DHTMessageTrackerEntry.cc src/DHTMessageTrackerEntry.h
        src/DHTNode.cc src/DHTNode.h
        src/DHTNodeLookupEntry.cc src/DHTNodeLookupEntry.h
        src/DHTNodeLookupTask.cc src/DHTNodeLookupTask.h
        src/DHTNodeLookupTaskCallback.cc src/DHTNodeLookupTaskCallback.h
        src/DHTPeerAnnounceCommand.cc src/DHTPeerAnnounceCommand.h
        src/DHTPeerAnnounceEntry.cc src/DHTPeerAnnounceEntry.h
        src/DHTPeerAnnounceStorage.cc src/DHTPeerAnnounceStorage.h
        src/DHTPeerLookupTask.cc src/DHTPeerLookupTask.h
        src/DHTPeerLookupTaskCallback.cc src/DHTPeerLookupTaskCallback.h
        src/DHTPingMessage.cc src/DHTPingMessage.h
        src/DHTPingReplyMessage.cc src/DHTPingReplyMessage.h
        src/DHTPingReplyMessageCallback.h
        src/DHTPingTask.cc src/DHTPingTask.h
        src/DHTQueryMessage.cc src/DHTQueryMessage.h
        src/DHTRegistry.cc src/DHTRegistry.h
        src/DHTReplaceNodeTask.cc src/DHTReplaceNodeTask.h
        src/DHTResponseMessage.cc src/DHTResponseMessage.h
        src/DHTRoutingTable.cc src/DHTRoutingTable.h
        src/DHTRoutingTableDeserializer.cc src/DHTRoutingTableDeserializer.h
        src/DHTRoutingTableSerializer.cc src/DHTRoutingTableSerializer.h
        src/DHTSetup.cc src/DHTSetup.h
        src/DHTTask.h
        src/DHTTaskExecutor.cc src/DHTTaskExecutor.h
        src/DHTTaskFactory.h
        src/DHTTaskFactoryImpl.cc src/DHTTaskFactoryImpl.h
        src/DHTTaskQueue.h
        src/DHTTaskQueueImpl.cc src/DHTTaskQueueImpl.h
        src/DHTTokenTracker.cc src/DHTTokenTracker.h
        src/DHTTokenUpdateCommand.cc src/DHTTokenUpdateCommand.h
        src/DHTUnknownMessage.cc src/DHTUnknownMessage.h
        src/ExtensionMessage.h
        src/ExtensionMessageFactory.h
        src/ExtensionMessageRegistry.cc src/ExtensionMessageRegistry.h
        src/HandshakeExtensionMessage.cc src/HandshakeExtensionMessage.h
        src/IndexBtMessage.cc src/IndexBtMessage.h
        src/IndexBtMessageValidator.cc src/IndexBtMessageValidator.h
        src/InitiatorMSEHandshakeCommand.cc src/InitiatorMSEHandshakeCommand.h
        src/LpdDispatchMessageCommand.cc src/LpdDispatchMessageCommand.h
        src/LpdMessage.cc src/LpdMessage.h
        src/LpdMessageDispatcher.cc src/LpdMessageDispatcher.h
        src/LpdMessageReceiver.cc src/LpdMessageReceiver.h
        src/LpdReceiveMessageCommand.cc src/LpdReceiveMessageCommand.h
        src/magnet.cc src/magnet.h
        src/MemoryBencodePreDownloadHandler.h
        src/MSEHandshake.cc src/MSEHandshake.h
        src/NameResolveCommand.cc src/NameResolveCommand.h
        src/Peer.cc src/Peer.h
        src/PeerAbstractCommand.cc src/PeerAbstractCommand.h
        src/PeerAddrEntry.cc src/PeerAddrEntry.h
        src/PeerChokeCommand.cc src/PeerChokeCommand.h
        src/PeerConnection.cc src/PeerConnection.h
        src/PeerInitiateConnectionCommand.cc src/PeerInitiateConnectionCommand.h
        src/PeerInteractionCommand.cc src/PeerInteractionCommand.h
        src/PeerListenCommand.cc src/PeerListenCommand.h
        src/PeerReceiveHandshakeCommand.cc src/PeerReceiveHandshakeCommand.h
        src/PeerSessionResource.cc src/PeerSessionResource.h
        src/PeerStorage.h
        src/PriorityPieceSelector.cc src/PriorityPieceSelector.h
        src/RangeBtMessage.cc src/RangeBtMessage.h
        src/RangeBtMessageValidator.cc src/RangeBtMessageValidator.h
        src/ReceiverMSEHandshakeCommand.cc src/ReceiverMSEHandshakeCommand.h
        src/RequestSlot.cc src/RequestSlot.h
        src/SeedCheckCommand.cc src/SeedCheckCommand.h
        src/SeedCriteria.h
        src/ShareRatioSeedCriteria.cc src/ShareRatioSeedCriteria.h
        src/SimpleBtMessage.cc src/SimpleBtMessage.h
        src/TimeSeedCriteria.cc src/TimeSeedCriteria.h
        src/TrackerWatcherCommand.cc src/TrackerWatcherCommand.h
        src/UDPTrackerClient.cc src/UDPTrackerClient.h
        src/UDPTrackerRequest.cc src/UDPTrackerRequest.h
        src/UnionSeedCriteria.cc src/UnionSeedCriteria.h
        src/UTMetadataDataExtensionMessage.cc src/UTMetadataDataExtensionMessage.h
        src/UTMetadataExtensionMessage.cc src/UTMetadataExtensionMessage.h
        src/UTMetadataPostDownloadHandler.cc src/UTMetadataPostDownloadHandler.h
        src/UTMetadataRejectExtensionMessage.cc src/UTMetadataRejectExtensionMessage.h
        src/UTMetadataRequestExtensionMessage.cc
        src/UTMetadataRequestExtensionMessage.h
        src/UTMetadataRequestFactory.cc src/UTMetadataRequestFactory.h
        src/UTMetadataRequestTracker.cc src/UTMetadataRequestTracker.h
        src/UTPexExtensionMessage.cc src/UTPexExtensionMessage.h
        src/ValueBaseBencodeParser.h
        src/XORCloser.h
        src/ZeroBtMessage.cc src/ZeroBtMessage.h)
endif (ENABLE_BITTORRENT)

if (ENABLE_METALINK)
    list(APPEND SRCS
        src/Metalink2RequestGroup.cc src/Metalink2RequestGroup.h
        src/MetalinkEntry.cc src/MetalinkEntry.h
        src/Metalinker.cc src/Metalinker.h
        src/MetalinkMetaurl.cc src/MetalinkMetaurl.h
        src/MetalinkParserController.cc src/MetalinkParserController.h
        src/MetalinkParserState.cc src/MetalinkParserState.h
        src/MetalinkParserStateImpl.cc src/MetalinkParserStateImpl.h
        src/MetalinkParserStateMachine.cc src/MetalinkParserStateMachine.h
        src/MetalinkParserStateV3Impl.cc src/MetalinkParserStateV3Impl.h
        src/MetalinkParserStateV4Impl.cc src/MetalinkParserStateV4Impl.h
        src/MetalinkPostDownloadHandler.cc src/MetalinkPostDownloadHandler.h
        src/MetalinkResource.cc src/MetalinkResource.h
        src/metalink_helper.cc src/metalink_helper.h)
endif (ENABLE_METALINK)

if (NOT HAVE_ASCTIME_R)
    list(APPEND SRCS src/asctime_r.c src/asctime_r.h)
endif (NOT HAVE_ASCTIME_R)

if (NOT HAVE_BASENAME)
    list(APPEND SRCS src/libgen.c src/libgen.h)
endif (NOT HAVE_BASENAME)

if (NOT HAVE_GETADDRINFO)
    list(APPEND SRCS src/getaddrinfo.c src/getaddrinfo.h)
endif (NOT HAVE_GETADDRINFO)

if (NOT HAVE_GAI_STRERROR)
    list(APPEND SRCS src/gai_strerror.c src/gai_strerror.h)
endif (NOT HAVE_GAI_STRERROR)

if (NOT HAVE_GETTIMEOFDAY)
    list(APPEND SRCS src/gettimeofday.c src/gettimeofday.h)
endif (NOT HAVE_GETTIMEOFDAY)

if (NOT HAVE_LOCALTIME_R)
    list(APPEND SRCS src/localtime_r.c src/localtime_r.h)
endif (NOT HAVE_LOCALTIME_R)

if (NOT HAVE_STRPTIME)
    list(APPEND SRCS src/strptime.c src/strptime.h)
endif (NOT HAVE_STRPTIME)

if (NOT HAVE_TIMEGM)
    list(APPEND SRCS src/timegm.c src/timegm.h)
endif (NOT HAVE_TIMEGM)

if (NOT HAVE_DAEMON)
    list(APPEND SRCS src/daemon.cc src/daemon.h)
endif (NOT HAVE_DAEMON)

if (HAVE_POLL)
    list(APPEND SRCS src/PollEventPoll.cc src/PollEventPoll.h)
endif (HAVE_POLL)

if (HAVE_PORT_ASSOCIATE)
    list(APPEND SRCS src/PortEventPoll.cc src/PortEventPoll.h)
endif (HAVE_PORT_ASSOCIATE)

if (HAVE_KQUEUE)
    list(APPEND SRCS src/KqueueEventPoll.cc src/KqueueEventPoll.h)
endif (HAVE_KQUEUE)

if (HAVE_LIBUV)
    list(APPEND SRCS src/LibuvEventPoll.cc src/LibuvEventPoll.h)
endif (HAVE_LIBUV)


if (ENABLE_LIBARIA2)
    list(APPEND SRCS
    src/ApiCallbackDownloadEventListener.cc src/ApiCallbackDownloadEventListener.h
    src/aria2api.cc src/aria2api.h
    src/KeepRunningCommand.cc src/KeepRunningCommand.h)
endif (ENABLE_LIBARIA2)

add_library(aria2 STATIC ${SRCS})

target_link_libraries(aria2 ${LIBRARIES})

target_compile_features(aria2 PUBLIC
    cxx_defaulted_functions
)

add_executable(aria2c src/main.cc)
target_link_libraries(aria2c aria2)
