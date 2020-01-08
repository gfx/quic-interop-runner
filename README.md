# Interop Test Runner

The Interop Test Runner aims to automatically generate an interop matrix by running multiple **test cases** using different QUIC implementations.

## Running the Interop Runner

The Interop Runner is written in Python 3. You'll need to install a few Python modules to run it:

```bash
python3 setup.py install
```

Furthermore, you need to install development version of Wireshark (version 3.1.0 or newer).

Run the interop tests:
```bash
python3 run.py
```

## Building a QUIC endpoint

To include your QUIC implementation in the Interop Runner, create a Docker image following the instructions for [setting up an endpoint in the quic-network-simulator](https://github.com/marten-seemann/quic-network-simulator), publish it on [Docker Hub](https://hub.docker.com) and add it to [implementations.py](implementations.py). Once your implementation is ready to interop, please send us a PR with this addition. Read on for more instructions on what to do within the Docker image.

Typically, a test case will require a server to serve files from a directory, and a client to download files. Different test cases will specify the behavior to be tested. For example, the Retry test case expects the server to use a Retry before accepting the connection from the client. All configuration information from the test framework to your implementation is fed into the Docker image using environment variables. The test case is passed into your Docker container using the `TESTCASE` environment variable. If your implementation doesn't support a test case, it MUST exit with status code 127. This will allow us to add new test cases in the future, and correctly report test failures und successes, even if some implementations have not yet implented support for this new test case.

The Interop Runner mounts the directory `/www` into your server Docker container. This directory will contain one or more randomly generated files. Your server implementation is expected to run on port 443 and serve files from this directory.
Equivalently, the Interop Runner mounts `/downloads` into your client Docker container. The directory is initially empty, and your client implementation is expected to store downloaded files into this directory. The URLs of the files to download are passed to the client using the environment variable `REQUESTS`, which contains one or more URLs, separated by a space.

After the transfer is completed, the client container is expected to exit with exit status 0. If an error occurred during the transfer, the client is expected to exit with exit status 1.
After completion of the test case, the Interop Runner will verify that the client downloaded the files it was expected to transfer, and that the file contents match. Additionally, for certain test cases, the Interop Runner will use the pcap of the transfer to verify that the implementations fulfilled the requirements of the test (for example, for the Retry test case, the pcap should show that a Retry packet was sent, and that the client used the Token provided in that packet).

## Logs

To facilitate debugging, the Interop Runner saves the log files to the logs directory. This directory is overwritten every time the Interop Runner is executed.

The log files are saved to a directory named `#server_#client/#testcase`. `output.txt` contains the console output of the interop test runner (which might contain information why a test case failed). The server and client logs are saved in the `server` and `client` directory, respectively. The `sim` directory contains pcaps recorded by the simulator.

## Test cases

The Interop Runner implements the following test cases. Unless noted otherwise, test cases use HTTP/0.9 for file transfers. More test cases will be added in the future, to test more protocol features. The name in parentheses is the value of the `TESTCASE` environment variable passed into your Docker container.

* **Version Negotiation** (`versionnegotiation`): This test case tests that a server sends a Version Negotiation packet in response to an unknown QUIC version number. The client should start a connection using an unsupported version number (it can use a reserved version number to do so), and should abort the connection attempt when receiving the Version Negotiation packet.
Currently disabled due to #20.

* **Handshake** (`handshake`): This test case tests the successful completion of the handshake. The client is expected to establish a single QUIC connection to the server and download one or multiple small files. Servers should not send a Retry packet in this test case.

* **Transfer** (`transfer`): This test case tests both flow control and stream multiplexing. The client should use small initial flow control windows for both stream- and connection-level flow control, such that the during the transfer of files on the order of 1 MB the flow control window needs to be increased. The client is exepcted to establish a single QUIC connection, and use multiple streams to concurrently download the files.

* **Retry** (`retry`): This test case tests that the server can generate a Retry, and that the client can act upon it (i.e. use the Token provided in the Retry packet in the Initial packet).

* **Resumption** (`resumption`): This test cases tests QUIC session resumption (without 0-RTT). The client is expected to establish a connection and download the first file. The server is expected to provide the client with a session ticket that allows it to resume the connection. After downloading the first file, the client has to close the connection, establish a resumed connection using the session ticket, and use this connection to download the remaining file(s).

* **HTTP3** (`http3`): This test case tests a simple HTTP/3 connection. The client is expected to download multiple files using HTTP/3. Files should be requested and transfered in parallel.
