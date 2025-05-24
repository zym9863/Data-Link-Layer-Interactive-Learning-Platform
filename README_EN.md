**English** | [中文](./README.md)

# Data Link Layer Interactive Learning Platform
**数据链路层交互式学习平台**

This is an interactive learning platform built with Vue 3 + TypeScript + Vite, focusing on visualization and simulation of data link layer protocols.

## 🚀 Project Overview

This platform provides two core learning modules to help students and networking enthusiasts understand the working principles of the data link layer:

### 📦 Ethernet Frame Structure Analyzer
- **Interactive Frame Construction**: Visualize the Ethernet frame encapsulation process
- **Real-time Field Analysis**: Click on frame fields to view detailed explanations
- **Preset Sample Data**: Includes common packets like HTTP requests, Ping packets, ARP requests
- **Dynamic Validation**: Real-time calculation and verification of CRC checksums
- **Hexadecimal Display**: View raw binary data of frames

### 🌐 CSMA/CD Protocol Simulator
- **Dynamic Network Topology**: Adjustable number of network nodes (2-6 nodes)
- **Collision Detection Simulation**: Realistic demonstration of carrier sensing and collision detection
- **Backoff Algorithm Visualization**: Visual representation of binary exponential backoff algorithm
- **Parameter Adjustment**: Configurable transmission speed, maximum backoff time, and other parameters
- **Real-time Status Monitoring**: Display current status and operations of each node

## 🛠️ Tech Stack

- **Frontend Framework**: Vue 3 (Composition API)
- **Development Language**: TypeScript
- **Build Tool**: Vite
- **Styling**: CSS3 (modern gradients, animation effects)

## 📁 Project Structure

```
src/
├── App.vue                      # Main application component
├── main.ts                      # Application entry point
├── components/
│   ├── EthernetFrameAnalyzer.vue # Ethernet frame analyzer component
│   └── CSMASimulator.vue         # CSMA/CD simulator component
└── assets/                      # Static resources
```

## 🚦 Quick Start

### Install Dependencies
```bash
npm install
```

### Development Mode
```bash
npm run dev
```

### Build for Production
```bash
npm run build
```

### Preview Build
```bash
npm run preview
```

## 🎯 Learning Objectives

Using this platform, learners can:

1. **Understand Ethernet Frame Structure**
   - Master the roles of preamble, frame header, data payload, and CRC checksum
   - Learn the meaning of MAC addresses and type fields
   - Understand minimum and maximum frame length limitations

2. **Master CSMA/CD Protocol**
   - Understand carrier sensing mechanism
   - Learn collision detection and handling procedures
   - Master binary exponential backoff algorithm

3. **Network Communication Principles**
   - Understand shared medium access control
   - Learn how network collisions occur and are resolved
   - Master basic Ethernet working principles

## 🎨 Key Features

- **🎭 Visual Teaching**: Intuitive demonstration of protocol operations through animations and graphics
- **🎮 Interactive Experience**: Support for parameter adjustment and real-time simulation
- **📚 Education-Friendly**: Suitable for classroom teaching and self-learning
- **💻 Modern Interface**: Responsive design supporting multiple devices

## 🤝 Contributing

Welcome to submit Issues and Pull Requests to help improve this project!

## 📄 License

This project is licensed under the MIT License.

---

🌟 If this project helps you, please give it a Star!
