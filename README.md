# Face Recognition alarm System
The aim of this project is to implement an alarm detection system. The main objective is to identify if the person entering through the main door is authorized or is an unknown intruder. In my case, this information is used by my home automation server, Home Assistant (HA).

# Hardware
The hardware used is the following:
- A server where all containers and moduls can run.
   In my case i use my Ugreen Nas with 8GB of RAM.
- Edge camera (TP-Link Tapo C220)
  In my case i use the TP-Link Tapo C220: the quality is 2K QHD (4MP) with an aperture of $f/2.0$; it has the native RTSP dual stream, with the main stream in high-resolution and the sub-stream in low-resolution.

# Software
The software architecture is service-oriented, based on Docker. All components are orchestrated with the docker-compose file over an internal virtual private network. The main components are:
- **Frigate**
  [Frigate](https://github.com/blakeblackshear/frigate) is an event-based Network Video Recorder with low latency. It is composed of different modules:
  - Multiplexer: *go2rtc* takes as input the RTSP stream of the camera and sends it to different clients. This is useful when we have more clients than camera channels.
  - It performs continuous frame-differencing motion detection.
  - It has an Object Detection model (i.e., *MobileDet* or *YOLO*) that classifies an entity into different base classes (e.g., Person, Dog, Car).
- **Double Take**
  [Double Take](https://github.com/jakowenko/double-take) is a middleware that orchestrates the entire pipeline in a computer vision environment.
  - It takes the correct input from the correct topic list in a broker. This is useful if we have more than one camera and we have more than one operation to do for each camera.
  - It asks for computation from one or more models and can perform a consensus operation (e.g., average) to increase the recall metrics. The challenge in this case will be to normalize multiple JSON results formatted in different ways.
  - It sends the result to one or more clients (e.g., HA).
- **CompreFace**
   [CompreFace](https://github.com/exadel-inc/CompreFace) is a modular stack based on a microservices architecture.
   - Front-end: It is a graphical interface to create projects, handle face collections, and manage authentication.
   - Database: It is an instance of PostgreSQL and implements a relational and vector database. It is used to collect the face information about the authenticated person.
   - Inference Engine: It is the CompreFace core that exposes a REST API and executes the deep learning models at runtime. The recognition pipeline is the following: it finds the bounding box of the face; it performs an alignment; it extracts the features, the embedding of 512 dimensions; it performs the matching and returns a JSON with the person and the maximum cosine similarity found in the database.

# Data Flow
The camera data flow must be transformed into an output that can be useful for a client. This process defines various phases:
1. Input Multiplexing
   The *go2rtc* module of *Frigate* behaves as a proxy: it receive frame from the camera using both two channel, the main-stream (high-resolution channel with a high bitrate) and the sub-stram (low-resolution channel with a low bitrate); if more then one client want to read the camera frame (e.g., live camera control from smartphone), have to ask to this component.
2. Hierarchical Detection
   - To conserve the CPU of the host device, continuous processing is performed only over the sub-stream. To implement this, Frigate uses the OpenCV library with the following pipeline: conversion to grayscale of two consecutive frames; difference of the pixel values; if the resulting frame is totally black, the two frames are equal.
   - Thanks to the detail above, if nobody enters the house, we don't call any AI models. Instead, if something changes in two different frames within the zone of interest (i.e., the zone monitored), Frigate extracts it and sends it to its own object detection model module: if the model decides that there is a person in the frame, Frigate initializes a trace with its tracking ID and publishes an event on a broker.
3. Orchestration
   Double Take intercepts the event published by Frigate, validates the camera and the zone of interest, and through an HTTP GET retrieves from Frigate the frame in high-definition from the main stream. Now the frame is sent to CompreFace.
4. Face Recognition
   CompreFace has the objective to recognize if the person who entered is authorized or not: *RetinaFace* isolates the face and detect 5 coordinates (eyes, nose, mouth corners); the backbone of the CNN *ArcFace* extracts the embeddings of 512 dimensions; computes the cosine similarity against all vectors in the database; send back to Double Take the maximum value found.
5. Consensus and client dispatch
   Double Take compares the result with a given threshold: if the confidence if less than the theshould, then the person is not authorized and it sends this information to the client (i.e., HA).