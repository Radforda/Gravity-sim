<template>
  
  <div class="gravity-simulation">
<!---------------------------- Simulation Panel------------------------->
    <div class="sim-panel">
        <div class="simulation-container">
          <div class="height-scale">
            <div v-for="i in 10" :key="i" class="scale-mark">
              {{ (100 - i * 10) }} m
            </div>
          </div>
          <div ref="simContainer" class="simulation-area">
            <!-- PIXI.js simulation will render here -->
            <div
              v-if="cursorPosition"
              class="cursor-height"
              :style="{ left: `${cursorPosition.x}px`, top: `${cursorPosition.y}px` }"
            >
              {{ cursorHeight }} m
            </div>
          </div>
        </div>
    </div>

<!---------------------------- Data Panel------------------------------->
    <div class="data-panel">
      
<!-------------------------- Controls Section --------------------------->
      <section class="item-details">
        <h2>Gravity Simulation</h2>
        <section class="controls">
          <button @click="startSimulation" :disabled="disableStart">Start Simulation</button>
          <button @click="resetSimulation">Reset Simulation</button>
        </section>

<!--------------- Item Selection ----------------------> 
        <section class="item-selection">
          <h3>Select Item</h3>
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
        <div class="stats-card">
          <div class="stat-row">
            <span class="stat-label">Mass:</span>
            <span class="stat-value">{{ selectedItemData.mass }} kg</span>
          </div>
          <div class="stat-row">
            <span class="stat-label">Drag:</span>
            <span class="stat-value">{{ selectedItemData.drag }}</span>
          </div>
          <div class="stat-row">
            <span class="stat-label">Area:</span>
            <span class="stat-value">{{ selectedItemData.area }} m²</span>
          </div>
          <div class="stat-row">
            <span class="stat-label">Volume:</span>
            <span class="stat-value">{{ selectedItemData.volume }} m³</span>
          </div>
        </div>
       
<!--------------- Environment Selection ------------------>
        <section class="environment-selection">
          <h3>Select Environment</h3>
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
        <div class="stats-card">
          <div class="stat-row">
            <span class="stat-label">Gravity:</span>
            <span class="stat-value">9.81 m/s</span>
          </div>
          <div class="stat-row">
            <span class="stat-label">Density:</span>
            <span class="stat-value">{{selectedEnvironment == "air" ? "1.225 kg/m³": "0 kg/m³"}}</span>
          </div>
        </div>
        <h3> Computed Properties</h3>
        <div class="stats-card">
          <div class="stat-row">
            <span class="stat-label">Weight:</span>
            <span class="stat-value">{{(selectedItemData.mass * 9.81).toFixed(3)}} N</span>
          </div>
          <div class="stat-row">
            <span class="stat-label">Buoyancy Force:</span>
            <span class="stat-value">{{(selectedEnvironment == "air" ? 1.225 * selectedItemData.volume * 9.81: 0).toFixed(3)}} N</span>
          </div>
        </div>
      </section>
    

<!-------------------------- Graphs --------------------------------------->
      <section class="graphs">
        <h3>Graphs</h3>
  
        <div class="graph-container">
          <canvas id="velocityGraph"></canvas>
        <canvas id="velocityGraph"></canvas>
      </div>
        
        <div class="graph-container">
        <canvas id="dragGraph"></canvas>
       </div>
        
       
        <div class="graph-container">
        <canvas id="positionGraph"></canvas>
      </div>
      </section>


<!---------------------------- Table Section--------------------------------->
      <section class="data-table">
        <h3>Collected Data</h3>
        <table>
          <thead>
            <tr>
              <th>Time (s)</th>
              <th>Velocity (m/s)</th>
              <th>Position (m)</th>
              <th>Fdrag (N)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(entry, index) in data" :key="index">
              <td>{{ entry.time }}</td>
              <td>{{ entry.velocity }}</td>
              <td>{{ entry.position }}</td>
              <td>{{ entry.Fdrag }}</td>
            </tr>
          </tbody>
        </table>
      </section>
    </div>
  </div>
</template>





<script>
import * as PIXI from "pixi.js";
import { ref, computed, onMounted, onUnmounted, watch } from "vue";
import Chart from "chart.js/auto";
import ballImage from "./assets/ball.png";
import balloonImage from "./assets/balloon.png";
import featherImage from "./assets/feather.png";
import backgroundImage from "./assets/background.png"; 

export default {
  setup() {
    let disableStart = ref(false);
    const simContainer = ref(null);
    const cursorPosition = ref({ x: 0, y: 0 }); 
    const cursorHeight = ref(0);
    const selectedItem = ref(1);
    let velocityChart = null;
    let dragChart = null;
    let positionChart = null;
    const selectedEnvironment = ref("air"); 
    const data = ref([]); 
    let pixiApp = null;
    let item = null;
    const gravity = 9.81; //9.81 m/s²
    const canvasWidth = 400;
    const canvasHeight = 700; 
    const simulationHeight = 100; // Maximum height in meters
    const scalingFactor = canvasHeight / simulationHeight; // Pixels per meter
 

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

    const selectedItemData = computed(() => {
      return items.value.find((item) => item.id === selectedItem.value);
    });
    const selectedItemName = computed(() => {
      const selected = items.value.find((item) => item.id === selectedItem.value);
      return selected ? selected.name : "None";
    });


    //---------------------------Initialize PIXI.js simulation-----------------------------
   

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



// ---------------------------------------------------------------------------------------------------
//-----------------------------  simulation logic  ---------------------------------------------------
// ---------------------------------------------------------------------------------------------------

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

    const startSimulation = () => {
      if(disableStart.value) return;
      disableStart.value = true 

      if (!pixiApp || !item) return;

      // Get the drag, buoyancy, mass, and bounce properties of the selected item
      const selectedItemData = items.value.find((item) => item.id === selectedItem.value);
      const Cdrag = selectedItemData.drag;
      const area = selectedItemData.area; // Cross-sectional area in m²
      const mass = selectedItemData.mass; // Mass in kg
      const volume = selectedItemData.volume; // Volume in m³
      const bounce = selectedItemData.bounce;


      // Initialize variables for simulation
      let velocity = 0;
      let elapsedTime = 0; // Track elapsed time for recording velocity
      let totalTime = 0; // Track total simulation time
      let positionY = selectedItemData ? selectedItemData.initialPosition / scalingFactor : canvasHeight / 2; // Initial position in meters
      let Fbuoyancy = selectedEnvironment.value === "air" ? 1.225 * volume * gravity: 0; // Buoyancy force in air (ρ * V * g)
      let Fdrag = 0; // Drag force
      let Weight =  mass * gravity; // Weight of the item in Newtons


      //constants for x oscillations
      let oscillationAngle = 180; // Angle for oscillation when falling
      const oscillationAmplitude = .5; // amplitude for oscillation when falling

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

        //------------ Calculate the time elapsed since the last frame---------------------

        const deltaTime = pixiApp.ticker.elapsedMS / 1000; // Convert elapsed time to seconds
        elapsedTime += deltaTime;
        totalTime += deltaTime;

        //-------------------Update the item's position and velocity-------------------

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


        // ------------------Add oscillating motion in the x-direction-----------------

        if (mass < 0.25 && !(item.y <= item.height / 2)  && selectedEnvironment.value === "air") {
          // Increment the angle for oscillation
            oscillationAngle += 2 * deltaTime; 
          // Oscillate in the x-direction
          item.x += Math.sin(oscillationAngle) * oscillationAmplitude; 
          //rotate the item
          item.rotation += Math.sin(oscillationAngle*2) * deltaTime; 
        }

        // ---------------Record Data for Table and Charts ---------------------
        if (elapsedTime >= 0.5) {
          // Record the data for table
          data.value.push({
            time: totalTime.toFixed(1),
            velocity: Math.abs(velocity.toFixed(2)),
            position: positionY.toFixed(2), // Add real-world position in meters
            Fbuoyancy: Fbuoyancy.toFixed(2), // Buoyancy force
            Fdrag: Fdrag.toFixed(3), // Drag force
            Weight: Weight.toFixed(3), // Weight of the item
          });
          elapsedTime = 0; // Reset elapsed time
        
          // record the chart data
          velocityChart.data.labels = data.value.map((entry) => entry.time); 
          velocityChart.data.datasets[0].data = data.value.map((entry) => entry.velocity); 
          velocityChart.update();
          dragChart.data.labels = data.value.map((entry) => entry.time); 
          dragChart.data.datasets[0].data = data.value.map((entry) => entry.Fdrag); 
          dragChart.update();
          positionChart.data.labels = data.value.map((entry) => entry.time); 
          positionChart.data.datasets[0].data = data.value.map((entry) => entry.position); 
          positionChart.update();
        }

        //--------------------------end the simulation-----------------------------

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
      velocityChart.data.labels = []; 
      velocityChart.data.datasets[0].data = []; 
      velocityChart.update(); 
      dragChart.data.labels = [];
      dragChart.data.datasets[0].data = [];
      dragChart.update();
      positionChart.data.labels = []; 
      positionChart.data.datasets[0].data = []; 
      positionChart.update();
      data.value = [];
      disableStart.value = false;
    };

    const destroySimulation = () => {
      if (pixiApp) {
        pixiApp.destroy(true, { children: true });
        pixiApp = null;
      }
    };

    watch(
      selectedItem,
      async (newItemId) => {
        const newItem = items.value.find((item) => item.id === newItemId);
        if (newItem) {
          resetSimulation()
        }
      },
    );
    watch(
      selectedEnvironment,
      async (newEnv) => {
        if (newEnv) {
          resetSimulation() 
        }
      },
    );


    const updateCursorHeight = (event) => {
      if (!simContainer.value) return;

      const rect = simContainer.value.getBoundingClientRect();
      const cursorX = event.clientX;
      const cursorY = event.clientY;

      // Check if the cursor is inside the simulation container
      const isInside = cursorX >= rect.left && cursorX <= rect.right && cursorY >= rect.top && cursorY <= rect.bottom;

      if (isInside) {
        cursorPosition.value = {
          x: cursorX - rect.left+ 15,
          y: cursorY - rect.top,
        };
        cursorHeight.value = ((canvasHeight - cursorPosition.value.y) / scalingFactor).toFixed(2);
      } else {
        cursorPosition.value = null; // Hide the cursor text
        cursorHeight.value = null; // Reset the height value
      }
    };


  const initializeGraphs = () => {
    const ctx = document.getElementById("velocityGraph").getContext("2d");
    velocityChart = new Chart(ctx, {
      type: "line",
      data: {
        labels: data.value.map((entry) => entry.time), // Time values
        datasets: [
          {
            label: "Velocity (m/s)",
            data: data.value.map((entry) => entry.velocity), // Velocity values
            borderColor: "#4caf50",
            fill: false,
          },
        ],
      },
      options: {
          plugins: {
            title: {
                  display: true,
                  text: 'Velocity vs Time',
              },
              legend: {
                  display: false, // This line removes the legend
              }
          },
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: {
            title: {
              display: true,
              text: "Time (s)",
            },
          },
          y: {
            title: {
              display: true,
              text: "Velocity (m/s)",
            },
          },
        },
      },
    });

    const ctxDrag = document.getElementById("dragGraph").getContext("2d");
    dragChart = new Chart(ctxDrag, {
      type: "line",
      data: {
        labels: data.value.map((entry) => entry.time), // Time values
        datasets: [
          {
            label: "Drag Force (N)",
            data: data.value.map((entry) => entry.Fdrag), // Drag force values
            borderColor: "#f44336",
            fill: false,
          },
        ],
      },
      options: {
        plugins: {
            title: {
                  display: true,
                  text: 'Drag vs Time',
              },
              legend: {
                  display: false, // This line removes the legend
              }
          },
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: {
            title: {
              display: true,
              text: "Time (s)",
            },
          },
          y: {
            title: {
              display: true,
              text: "Drag Force (N)",
            },
          },
        },
      },
    });

    const ctxPosition = document.getElementById("positionGraph").getContext("2d");
    positionChart = new Chart(ctxPosition, {
      type: "line",
      data: {
        labels: data.value.map((entry) => entry.time), // Time values
        datasets: [
          {
            label: "Position (m)",
            data: data.value.map((entry) => entry.position), // Position values
            borderColor: "#2196f3",
            backgroundColor: "rgba(33, 150, 243, 0.2)",
            fill: false,
          },
        ],
      },
      options: {
        plugins: {
            title: {
                  display: true,
                  text: 'Position vs Time',
              },
              legend: {
                  display: false, // This line removes the legend
              }
          },
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: {
            title: {
              display: true,
              text: "Time (s)",
            },
          },
          y: {
            title: {
              display: true,
              text: "Position (m)",
            },
          },
        },
      },
    });
  };

    onMounted(() => {
      initializeGraphs();
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
      if (velocityChart) {
        velocityChart.destroy();
      }
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
      selectedItemData, 
      startSimulation,
      resetSimulation,
      cursorPosition,
      cursorHeight,
      disableStart

    };
  },
};
</script>






<style>
.gravity-simulation {
  display: flex;
  justify-content: space-between;
  padding: 20px;
  flex-wrap: wrap;
}

.sim-panel {
  width: 400px; 
  height: 700px; 
  display: flex;
  justify-content: center;
  align-items: center;
}

.simulation-area {
  border: 1px solid #ccc;
  width: 100%; 
  height: 100%; 
}

.data-panel {
  flex: 1; 
  margin-top: 0; 
  padding: 10px;
  border: 1px solid #ccc;
  background-color: #f9f9f9;
  font-size: 14px;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: space-evenly;
}

.item-details h2 {
  margin-bottom: 10px;
}

.item-details p {
  margin: 5px 0;
}

.item-selection,
.environment-selection,
.data-table,
.controls {
  margin-bottom: 20px;
}


.item-options {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-evenly;
}

.item-option {
  margin-right: 10px;
  cursor: pointer;
  text-align: center;
}

.item-option img {
  width: 40px; 
  height: 40px;
  object-fit: contain;;
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
  justify-content: space-evenly;
}

.environment-option {
  margin-right: 10px;
  cursor: pointer;
}

.cursor-height {
  position: absolute;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px;
  border-radius: 5px;
  font-size: 12px;
}

.simulation-container {
  position: relative;
  width: 100%;
  height: 100%;
}

.height-scale {
  position: absolute;
  left: 15px;
  top: 0;
  bottom: 0;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  color: #333;
}

.scale-mark {
  height: 10%;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  position: relative;
}

.scale-mark::after {
  content: "";
  position: absolute;
  left: -15px; 
  width: 10px; 
  height: 2px; 
  background-color: #333; 
}

.stats-card {
  background-color: #fff;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.stat-row {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.stat-label {
  font-weight: bold;
  margin-right: 5px;
}

.stat-value {
  color: #333;
}

.graph-container {
  position: relative;
  width: 300px;
  height: 200px; 
}

.controls button {
  background-color: darkblue; 
  color: white; 
  border: none; 
  border-radius: 4px; 
  padding: 8px 16px; 
  font-size: 14px; 
  cursor: pointer; 
  transition: background-color 0.2s; 
  margin-right: 10px;
}


.controls button:last-child {
  margin-right: 0; 
}

.controls button:not(:disabled):hover {
  background-color: #0056b3; 
}

.controls button:not(:disabled):active {
  background-color: #003f7f;
}
button:disabled {
  background-color: #ccc;
  cursor: not-allowed; 
}
</style>