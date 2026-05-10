1: Container Communication And Types of Communication
2: Container To WWW Communication
 eg : docker run -it alpine sh
3: Container To Local Host Machine Communication
 eg : docker run -it --network host alpine sh
4: Container To Container Communication
    eg : docker run -it --network container:container_name alpine sh


5: How Docker Networking Simulates Real-World Network Internally

6: Creating A Container And Communicating To Web (WWW)