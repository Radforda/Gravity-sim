<template>
  <h1>Gravity Simulation</h1>
  <div class="gravity-simulation">

    <!-- Left Section: Item Selection and Table -->
    
      <div class="left-panel">
        <div class="top-row">
        <div  class="left-column">
          <!-- Control Buttons -->
          <section class="controls">
            <button @click="startSimulation">Start Simulation</button>
            <button @click="resetSimulation">Reset Simulation</button>
        
          </section>
          <!-- Item Selection Section -->
          <section class="item-selection">
            <h2>Select Item</h2>
            <div class="item-options">
              <label
                v-for="item in items"
                :key="item.id"
                class="item-option"
              >
                <input
                  v-model="selectedItem"
                  type="radio"
                  :value="item.id"
                  class="hidden-radio"
                />
                <img
                  :src="item.image"
                  :alt="item.name"
                  :class="{ selected: selectedItem === item.id }"
                />
                <p>{{ item.name }}</p>
              </label>
            </div>
          </section>
          <!-- Environment Selection Section -->
          <section class="environment-selection">
            <h2>Select Environment</h2>
            <div class="environment-options">
              <label class="environment-option">
                <input
                  v-model="selectedEnvironment"
                  type="radio"
                  value="air"
                />
                Air
              </label>
              <label class="environment-option">
                <input
                  v-model="selectedEnvironment"
                  type="radio"
                  value="vacuum" 
                />
                Vacuum
              </label>
            </div>
          </section>
        </div>
        <div class="right-column">
            <img class="forces-image" src="./assets/forces.png" alt="Forces acting on the object" />
        </div>
    </div>

      <!-- Table Section -->
      <section class="data-table">
        <h2>Time, Velocity, and Position</h2>
        <table>
          <thead>
            <tr>
              <th>Time (s)</th>
              <th>Velocity (m/s)</th>
              <th>Position (m)</th>
              <th>Fbuoy (N)</th>
              <th>Fdrag (N)</th>
              <th>Weight (N)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(entry, index) in data" :key="index">
              <td>{{ entry.time }}</td>
              <td>{{ entry.velocity }}</td>
              <td>{{ entry.position }}</td>
              <td>{{ entry.Fbuoyancy }}</td>
              <td>{{ entry.Fdrag }}</td>
              <td>{{ entry.Weight }}</td>
            </tr>
          </tbody>
        </table>
      </section>
    </div>

    <!-- Right Section: Simulation -->
    <div class="right-panel">
      <section class="simulation">
        <div ref="simContainer" class="simulation-area">
          <!-- PIXI.js simulation will render here -->
          <div
            class="cursor-height"
            :style="{ left: `${cursorPosition.x}px`, top: `${cursorPosition.y}px` }"
          >
            {{ cursorHeight }} m
          </div>
        </div>
      </section>
    </div>
  </div>
  <div>
    <h3>Governing Equations:</h3>
    <strong>1. Net Force:</strong> The net force acting on the object is calculated as:<br>
    F = Weight - Fdrag - Fbuoyancy<br>
    Where:<br>
    Weight = m * g (force due to gravity)<br>
    Fdrag = 0.5 * Cd * ρ * A * velocity² (drag force, dependent on velocity)<br>
    Fbuoyancy = ρ * Volume * g (buoyancy force, upward force due to displaced air)<br>
    <br>
    <strong>2.Acceleration:</strong> Using Newton's second law:<br>
    acceleration = F / m<br>
    Substituting the net force:<br>
    acceleration = g - (Fdrag / m) + (Fbuoyancy / m)<br>
    <br>
    <strong>3. Velocity Update:</strong> The velocity is updated using the kinematic equation:<br>
    velocity = velocity(at time 0) + (acceleration * delta_time)<br>
    This accounts for the cumulative effect of forces over time.<br>

    <a href="https://www1.grc.nasa.gov/beginners-guide-to-aeronautics/falling-object-with-air-resistance/">NASA article for reference</a><br>

    
  </div>
</template>

<script>
import * as PIXI from "pixi.js";
import { ref, computed, onMounted, onUnmounted, watch } from "vue";
import ballImage from "./assets/ball.png";
import balloonImage from "./assets/balloon.png";
import featherImage from "./assets/feather.png";
import backgroundImage from "./assets/background.png"; 

export default {
  setup() {
    const simContainer = ref(null);
    const cursorPosition = ref({ x: 0, y: 0 }); 
    const cursorHeight = ref(0);
    const selectedItem = ref(1); 
    const items = ref([
      { id: 1, 
        name: "Ball", 
        image: ballImage, 
        drag: 0.24, //Cd based on Nasa soccer ball drag coefficient
        mass: .45, // (kg) for a standard soccer ball
        area: .037,// (m²) cross-sectional area of the soccer ball 
        volume: 0.004, // (m³) estimated volume of the soccer ball r=.11m
        scale: 0.02,
        initialPosition: 350,  
        bounce: 0.4 }, 
      { id: 2, 
        name: "Balloon", 
        image: balloonImage, 
        drag: 0.20, //Cd based on Nasa soccer ball drag coefficient assuming similar for balloon https://www.grc.nasa.gov/www/k-12/airplane/socdrag.html
        mass: 0.0025 , // (kg) for a small balloon
        area: .0037, // (m²) cross-sectional area of the balloon
        volume: 0.004, // (m³) estimated volume of the balloon r=.1m
        scale: 0.025,
        initialPosition: 350, 
        bounce: 0.2 }, 
      { id: 3, 
        name: "Feather", 
        image: featherImage, 
        drag: .75, //Cd estimated
        mass: 0.005, // (kg) for a medium feather
        area: .005, // (m²) estimated for medium 6in feather
        volume: 0.0001, // (m³) estimated volume of the feather
        scale: 0.0075,
        initialPosition: 350, 
        bounce: 0.1 }, 
    ]);

    //This sim is based off the principles of falling objects outlined here in the NASA article:
    //https://www1.grc.nasa.gov/beginners-guide-to-aeronautics/falling-object-with-air-resistance/
    //F = mg-Fdrag-Fbuoyancy
    //a = F/m
    //a = g - (Fdrag/m) + (Fbuoyancy/m)
    //v = v0 + at
    //split up for easier calculations in different environments with different drag and buoyancy forces
    //v = v0 + g*t - (Fdrag/m)*t + (Fbuoyancy/m)*t

    //Fdrag = 1/2*Cd*ρ*A*velocity²
    //Fbuoyancy = ρ*Volume*g
 
    const selectedEnvironment = ref("air"); 
    const data = ref([]); 
    let pixiApp = null;
    let item = null;
    const gravity = 9.81; //9.81 m/s²

    // Canvas and simulation parameters
    const canvasWidth = 400;
    const canvasHeight = 700; 
    const simulationHeight = 100; // Maximum height in meters
    const scalingFactor = canvasHeight / simulationHeight; // Pixels per meter

    // Computed property for the name of the selected item
    const selectedItemName = computed(() => {
      const selected = items.value.find((item) => item.id === selectedItem.value);
      return selected ? selected.name : "None";
    });

    const initializeSimulation = async (itemImage) => {
      if (!simContainer.value) return;

      // Destroy the previous PIXI application if it exists
      if (pixiApp) {
        pixiApp.destroy(true, { children: true });
        pixiApp = null;
      }

      // Create PIXI.js application
      pixiApp = new PIXI.Application();
      await pixiApp.init({
        width: canvasWidth,
        height: canvasHeight, 
        backgroundAlpha: 0,// Fully transparent background
      });

      // Append the PIXI.js canvas to the simulation container
      simContainer.value.appendChild(pixiApp.canvas);

      // Preload the assets
      await PIXI.Assets.load(itemImage);
      await PIXI.Assets.load(backgroundImage);

      // Set the background
      const backgroundTexture = PIXI.Texture.from(backgroundImage);
      const background = new PIXI.Sprite(backgroundTexture);
      background.anchor.set(0.5);
      background.x = pixiApp.renderer.width / 2;
      background.y = pixiApp.renderer.height / 2;
      background.width = pixiApp.renderer.width;
      background.height = pixiApp.renderer.height;
      pixiApp.stage.addChild(background);

      // Add selected object 
      const texture = PIXI.Texture.from(itemImage);
      item = new PIXI.Sprite(texture);
      item.anchor.set(0.5);
      item.x = pixiApp.renderer.width / 2;

      // Scale the item based on the selected item's scale property
      const selectedItemData = items.value.find((item) => item.id === selectedItem.value);
      const scaleFactor = selectedItemData ? selectedItemData.scale : 0.025; 
      item.scale.set(scaleFactor);

      // Set the initial position based on the item's `initialPosition` property
      item.y = selectedItemData
        ? canvasHeight - selectedItemData.initialPosition - item.height / 2
        : pixiApp.renderer.height / 2;
     
      pixiApp.stage.addChild(item);
    };

    const startSimulation = () => {
      if (!pixiApp || !item) return;

      // Get the drag, buoyancy, mass, and bounce properties of the selected item
      const selectedItemData = items.value.find((item) => item.id === selectedItem.value);
      const Cdrag = selectedItemData ? selectedItemData.drag : 0;
      const area = selectedItemData ? selectedItemData.area : 0.01; // Cross-sectional area in m²
      const mass = selectedItemData ? selectedItemData.mass : 1.0; // Mass in kg
      const volume = selectedItemData ? selectedItemData.volume : 0.01; // Volume in m³
      const bounce = selectedItemData ? selectedItemData.bounce : 0;
      console.log("selectedItemData:", selectedItemData)

      //constants for x oscillations
      let oscillationAngle = 180; // Angle for oscillation when falling
      const oscillationAmplitude = .5; // amplitude for oscillation when falling

      // Initialize variables for simulation
      let velocity = 0;
      let elapsedTime = 0; // Track elapsed time for recording velocity
      let totalTime = 0; // Track total simulation time
      let positionY = selectedItemData ? selectedItemData.initialPosition / scalingFactor : canvasHeight / 2; // Initial position in meters
      let Fbuoyancy = selectedEnvironment == "air" ? 1.225 * volume * gravity: 0; // Buoyancy force in air (ρ * V * g)
      let Fdrag = 0; // Drag force
      let Weight =  mass * gravity; // Weight of the item in Newtons
      // Record initial data for time = 0
      data.value.push({
        time: totalTime.toFixed(1), // Time = 0
        velocity: velocity.toFixed(2), // Initial velocity = 0
        position: positionY.toFixed(2), // Initial position
        Fbuoyancy: Fbuoyancy.toFixed(2), // Initial buoyancy force
        Fdrag: Fdrag.toFixed(3), // Initial drag force
        Weight: Weight.toFixed(3), // Initial weight
      });

      // Start the simulation
      pixiApp.ticker.add(() => {

        // Calculate the time elapsed since the last frame
        const deltaTime = pixiApp.ticker.elapsedMS / 1000; // Convert elapsed time to seconds
        elapsedTime += deltaTime;
        totalTime += deltaTime;

        // Apply gravity (pulls downward, increases velocity positively)
        velocity += gravity * deltaTime; // m/s

        // Apply buoyancy (pushes upward, decreases velocity)
        if (selectedEnvironment.value === "air") {
          velocity -= (Fbuoyancy/mass) * deltaTime;
        }

        // Apply drag (acts opposite to velocity)
        if (selectedEnvironment.value === "air") {
          Fdrag = 0.5 * Cdrag * 1.225 * area * velocity * velocity; // Fdrag = 1/2*Cd*ρ*A*v²
          const direction = velocity > 0 ? -1 : 1; // Determine direction of drag force
          velocity += direction * (Fdrag/mass) * deltaTime; 
          console.log("drag:", velocity)
        }

        // Update the item's real-world position (in meters)
        positionY -= velocity * deltaTime; 

        //Handle ground collision
        if (positionY <= 0) {
          positionY = 0; 
          // Reverse velocity and apply bounce factor
          velocity = -velocity * bounce;
          // Stop bouncing if velocity is too small
          if (Math.abs(velocity) < 0.1) {
            velocity = 0;
          }
        }

        // Convert the real-world position to canvas position (in pixels)
        item.y = canvasHeight - positionY * scalingFactor - item.height / 2;

        // Add oscillating motion in the x-direction if mass is less than 0.75 to simulate light objects uneven drag 
        if (mass < 0.25 && !(item.y <= item.height / 2)  && selectedEnvironment.value === "air") {
          // Increment the angle for oscillation
            oscillationAngle += 2 * deltaTime; 
          // Oscillate in the x-direction
          item.x += Math.sin(oscillationAngle) * oscillationAmplitude; 
          //rotate the item
          item.rotation += Math.sin(oscillationAngle*2) * deltaTime; 
        }

        // Record velocity and position every half second
        if (elapsedTime >= 0.5) {
          data.value.push({
            time: totalTime.toFixed(1),
            velocity: Math.abs(velocity.toFixed(2)),
            position: positionY.toFixed(2), // Add real-world position in meters
            Fbuoyancy: Fbuoyancy.toFixed(2), // Buoyancy force
            Fdrag: Fdrag.toFixed(3), // Drag force
            Weight: Weight.toFixed(3), // Weight of the item
          });
          elapsedTime = 0; // Reset elapsed time
        }

        // Stop the simulation when the item comes to rest
        if (velocity === 0 && positionY === 0) {
          pixiApp.ticker.stop(); 
        }

        // Stop the simulation when the item is fully off the canvas (above the top boundary)
        if (item.y < -item.height / 2) {
        pixiApp.ticker.stop(); 
        }

      });
    };

    const resetSimulation = () => {
        destroySimulation();
        const item = items.value.find((item) => item.id === selectedItem.value);
        initializeSimulation(item.image);
      // Clear the `data` array
      data.value = [];
    };

    const destroySimulation = () => {
      if (pixiApp) {
        pixiApp.destroy(true, { children: true });
        pixiApp = null;
      }
    };

    // Watch for changes to the selected item and update the simulation
    watch(
      selectedItem,
      async (newItemId) => {
        const newItem = items.value.find((item) => item.id === newItemId);
        if (newItem) {
            destroySimulation();
            await initializeSimulation(newItem.image);
            data.value = [];
        }
      },
    );

    const updateCursorHeight = (event) => {
      if (!simContainer.value) return;
      const rect = simContainer.value.getBoundingClientRect();
      cursorPosition.value = {
        x: event.clientX,
        y: event.clientY - rect.top,
      };
      cursorHeight.value = ((canvasHeight - cursorPosition.value.y) / scalingFactor).toFixed(2);
    };

    onMounted(() => {
      const defaultItem = items.value.find((item) => item.id === selectedItem.value);
      if (defaultItem) {
        initializeSimulation(defaultItem.image); 
      }
      if (simContainer.value) {
        simContainer.value.addEventListener("mousemove", updateCursorHeight);
      }
    });

    onUnmounted(() => {
      destroySimulation();
      if (simContainer.value) {
        simContainer.value.removeEventListener("mousemove", updateCursorHeight);
      }
    });

    return {
      simContainer,
      selectedItem,
      selectedEnvironment, 
      items,
      data,
      selectedItemName, 
      startSimulation,
      resetSimulation,
      cursorPosition,
      cursorHeight,

    };
  },
};
</script>

<style>
.gravity-simulation {
  display: flex;
  justify-content: space-between;
  padding: 20px;
}

.left-panel {
  flex: 1;
  margin-right: 20px;
}

.right-panel {
  flex: 2;
}

.forces-image {
  width: 150px; 
  height: auto;
  margin-bottom: 10px;
}
.item-selection,
.environment-selection,
.data-table,
.controls {
  margin-bottom: 20px;
}
.top-row{
  display: flex;
  justify-content: space-between;
}
.left-column {
  flex-grow: 2;
}
.right-column {
  flex-grow: 0;
  max-width: 120px; /* Set a maximum width for the column */
  display: flex;
  justify-content: center;
  align-items: center;

}
.item-options {
  display: flex;
  flex-wrap: wrap;
}

.item-option {
  margin-right: 10px;
  cursor: pointer;
  text-align: center;
}

.item-option img {
  width: 50px; 
  height: 50px;
  border: 2px solid transparent;
  border-radius: 5px;
  transition: border-color 0.3s;
}

.item-option img.selected {
  border-color: blue;
}

.hidden-radio {
  display: none;
}

.environment-options {
  display: flex;
}

.environment-option {
  margin-right: 10px;
  cursor: pointer;
}

.simulation-area {
  border: 1px solid #ccc;
  width: 400px; 
  height: 700px;
}

.data-table {
  height: 380px; 
  overflow-y: auto; 
  border: 1px solid #ccc;
  padding: 10px;
  background-color: #f9f9f9;
}

.cursor-height {
  position: absolute;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px;
  border-radius: 5px;
  font-size: 12px;
}
</style>