cmake_minimum_required(VERSION 3.10)
project(ReactorHttp)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 添加 pthread 库
find_package(Threads REQUIRED)

set(SOURCES
    main.cpp
    Buffer.cpp
    Channel.cpp
    Dispatcher.cpp
    EpollDispatcher.cpp
    EventLoop.cpp
    HttpRequest.cpp
    Httpresponse.cpp
    PollDispatcher.cpp
    SelectDispatcher.cpp
    TcpConnection.cpp
    TcpServer.cpp
    ThreadPool.cpp
    WorkerThread.cpp
)

add_executable(reactor_http ${SOURCES})
target_link_libraries(reactor_http Threads::Threads)