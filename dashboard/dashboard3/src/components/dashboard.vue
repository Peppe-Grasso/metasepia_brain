<template>
  <v-container fluid class="fill-height" style="width:90vw;" >
    <v-col cols="12" md="7">
      <v-container style="height:100%; overflow:hidden">
          <img v-bind:src="'data:image/jpeg;base64,' + imageBytes1"
          style="border-radius: 2%; height: 100%; width: 100%; object-fit:cover"
          />
      </v-container>
    </v-col>   
    <v-col cols="12" md="5">
        <v-container >
          <v-card>
            <template v-slot:text>
              <pre>
voltage : {{ Math.round(average_voltage * 100) / 100 }}
charge  : {{ Math.floor(average_charge) }}%
              </pre>
              <v-progress-linear 
                  v-model="average_charge"
                  color="primary"
                  rounded="pill"
              ></v-progress-linear>
            </template>
          </v-card>
        </v-container>
      
      <v-col cols="12" md="5">
        
      </v-col>
        <v-container>
          <v-card>
            <template v-slot:text>
              <pre>
surge   : {{Math.round(surge * 100) / 100}} 
sway    : {{Math.round(sway * 100) / 100}}
pitch   : {{ Math.round(pitch * 100) / 100 }}
yaw     : {{ Math.round(yaw * 100) / 100 }}

amplitude         : {{amplitude}}
speed_multiplier  : {{Math.round(speed_multiplier * 100) / 100}}
              </pre>
              <v-row justify="end">
                <div style="padding-right: 30px; padding-left: 30px; padding-bottom: 30px; padding-top: 30px">
                  <Joystick
                  :size="75"
                  base-color="#494949"
                  stick-color="#FF7C0A"
                  :throttle="50"
                  :externalX=sway
                  :externalY=surge
                  />
                </div>
                <div style="padding-right: 30px; padding-left: 30px; padding-bottom: 30px; padding-top: 30px">
                  <Joystick
                  :size="75"
                  base-color="#494949"
                  stick-color="#FF7C0A"
                  :throttle="50"
                  :externalX=yaw
                  :externalY=pitch
                  />
                </div> 
              </v-row>
            </template>
          </v-card>
        </v-container>
      <v-container>
        <v-card>
          <template v-slot:text>
            <pre>
mode : {{ mode }}
            </pre>
          </template>
        </v-card>
      </v-container>
    </v-col>
  </v-container>
</template>

<script setup>
  import Joystick from './joysticks_bep.vue'
</script>

<script>
import ROSLIB from "roslib"

const ros = new ROSLIB.Ros({ 
url: "ws://192.168.194.52:9090" 
});

const battery_sub = new ROSLIB.Topic({
ros: ros,
name: "/BatteryState",
messageType: "custom_msgs/BatteryState"
});

const camera_sub = new ROSLIB.Topic({
ros: ros,
name: "/camera/camera/color/image_raw/compressed",
messageType: "sensor_msgs/msg/CompressedImage"
})

const cmd_sub = new ROSLIB.Topic({
ros: ros,
name: "/CMD",
messageType: "custom_msgs/CMD"
})

export default {
name: 'App',
components: {
},
data: function () {
  return {
    imageBytes1: "",
    imageBytes2: "",
    voltage: 0,
    charge: 0,
    surge: 0.0,
    sway: 0.0,
    pitch: 0.0,
    yaw: 0.0,
    amplitude: 0,
    speed_multiplier: 0,
    
    voltage_window: [],
    charge_window: [],
    
    average_voltage: 0,
    average_charge: 0,

    mode: "",
    dms: ""
  };
},
mounted() {
  this.init();
  document.title = "Metasepia Dashboard";
},
methods: {
  init: function () {
    ros.on("connection", function() {
      console.log("connected to websocket server.");
    });
    ros.on("error", function(error) {
      console.log("error connecting to websocket server: ", error);
    });
    ros.on("close", function() {
      console.log("connection to websocket server closed.");
    });

    var self = this;

    battery_sub.subscribe((message) => {
      console.log('Received message on ' + battery_sub.name + ': ' + message.voltage + ': ' + message.charge);
      self.voltage = message.voltage;
      self.add_voltage(message.voltage);

      self.charge = message.charge;
      self.add_charge(message.charge);
    });

    camera_sub.subscribe((message) => {
      console.log('Received message on ' + camera_sub.name);
      self.imageBytes1 = message.data;
    });

    cmd_sub.subscribe((message) => {
      console.log('Received message on ' +  cmd_sub.name + ': ' + message.mode + ': ' + message.surge + ': ' + message.sway + ': ' + message.pitch + ': ' + message.yaw + ': ' + message.amplitude + ': ' + message.speed_multiplier);
      self.surge = message.surge;
      self.sway = message.sway;
      self.pitch = message.pitch;
      self.yaw = message.yaw;
      self.amplitude = message.amplitude;
      self.speed_multiplier = message.speed_multiplier;
      
      if (message.mode == 0) {
        self.mode = "standby";
      } else if (message.mode == 1) {
        self.mode = "aquatic";
      } else if (message.mode == 2) {
        self.mode = "land";
      } else {
        self.mode = "off";
      }

      self.add_surge(message.surge);
      self.add_sway(message.sway);
    });

  },
  add_data(key, data, maxWindowSize) {
    // Ensure the window array exists for the given key
    if (!this[key + '_window']) {
      this[key + '_window'] = [];
    }
    
    // If the window exceeds the max size, remove the oldest entry
    if (this[key + '_window'].length >= maxWindowSize) {
      this[key + '_window'].shift();
    }

    // Push the new data into the window
    this[key + '_window'].push(data);
    
    // Calculate the average for the window
    this.calculate_average(key);
  },
  
  // General method to calculate the average for any key's window
  calculate_average(key) {
    const window = this[key + '_window'];
    
    if (window && window.length > 0) {
      this['average_' + key] = window.reduce((partialSum, a) => partialSum + a, 0) / window.length;
    }
  },
  add_voltage(voltage) {
    this.add_data('voltage', voltage, 50);
  },
  add_charge(charge) {
    this.add_data('charge', charge, 100);
  }
}
}
</script>