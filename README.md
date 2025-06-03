# Gravity Simulation

A physics-based simulation built with Vue 3 and PIXI.js that demonstrates the effects of gravity, drag, buoyancy, and collisions on different objects in various environments.

## Features

- **Object Selection**: Choose from a ball, balloon, or feather, each with unique physical properties.
- **Environment Selection**: Simulate in air or vacuum to observe the effects of drag and buoyancy.
- **Real-Time Simulation**: Visualize the motion of objects with accurate physics calculations.
- **Data Table**: View time, velocity, and position data in real-time.
- **Interactive Cursor**: Hover over the simulation area to see the height at any point.

## Installation

    1. Clone the repository:
        git clone https://github.com/radforda/gravity-sim.git
        cd gravity-sim
    2. Install dependencies:
        npm install
    3. Start the development server:
        npm run dev

## Build for Production

To build the project for production, run:
    npm run build
The output will be in the dist directory.

## Deployment

This project is configured to deploy to AWS amplify through the aws amplify console. Push changes to the main branch to trigger the deployment. Here is the link to the running app:
https://main.d2j0r9hxyxtwtt.amplifyapp.com/

## Technologies Used

Vue 3: Frontend framework for building the user interface.
PIXI.js: 2D rendering engine for the simulation.
Vite: Build tool for fast development and production builds.

## License

This project is licensed under the MIT License. 

