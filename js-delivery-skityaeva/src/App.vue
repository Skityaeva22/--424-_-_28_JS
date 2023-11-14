<script setup>
import {ref} from "vue";
import { Chart, registerables } from 'chart.js';

let hrs = ref(0)
let min = ref(0)
let sec = ref(0)

let orderTime = 0
let orderNumber = 1

let orderHistory = ref([])

let working = false

Chart.register(...registerables);
let moneyChart;
let profitChart;

const data = ref([
  {time: 0, income: 0, expenses: 0, profit: 0, netProfit: 0}
])


let finance = ref({
      income: 0,
      expenses: 0,
      fuelCost: 0,
      serviceCost: 0,
      repairCost: 0,
      salaryCost: 0,
      profit: 0,
      tax: 0,
      netProfit: 0
    }
)

let tab = ref()

let cargos = ref([])

let routes = ['Москва', 'Санкт-Петербург', 'Казань']

let cars = ref([
  {
    name: 'Mercedes-Benz NG',
    volume: 70.0,
    weight: 50.0,
    usedVolume: 0.0,
    usedWeight: 0.0,
    time: 0,
    working: false,
    broken: false,
    delivering: false,
    items: [],
    routes: [
      {
        city: "Казань",
        cargos: 0
      },
      {
        city: "Москва",
        cargos: 0
      },
      {
        city: "Санкт-Петербург",
        cargos: 0
      }
    ]
  },
  {
    name: 'Kenworth W900',
    volume: 120.0,
    weight: 100.0,
    usedVolume: 0.0,
    usedWeight: 0.0,
    time: 0,
    working: false,
    broken: false,
    items: [],
    routes: [
      {
        city: "Казань",
        cargos: 0
      },
      {
        city: "Москва",
        cargos: 0
      },
      {
        city: "Санкт-Петербург",
        cargos: 0
      }
    ]
  }
])

let auto = false

const prepareDelivery = (car) => {
  for (let i in car.items) {
    switch (car.items[i].destination) {
      case car.routes[0].city:
        car.routes[0].cargos++
        break
      case car.routes[1].city:
        car.routes[1].cargos++
        break
      case car.routes[2].city:
        car.routes[2].cargos++
        break
    }
  }
  car.working = true
  deliveryTimer(car)
}

const deliveryTimer = (car) => {
  console.log(car.delivering)
  if (!car.broken) {
    if (!car.delivering) {
      if (car.routes[0].cargos !== 0) {
        car.delivering = true
        progressTimer(car, 2, 2)
        setTimeout(() => delivery(car, car.routes[0].city), 2000)
        setTimeout(() => deliveryTimer(car), 1000)
      } else {
        if (car.routes[1].cargos !== 0) {
          car.delivering = true
          progressTimer(car, 8, 8)
          setTimeout(() => delivery(car, car.routes[1].city), 8000)
          setTimeout(() => deliveryTimer(car), 1000)
        } else {
          if (car.routes[2].cargos !== 0) {
            car.delivering = true
            progressTimer(car, 10, 10)
            setTimeout(() => delivery(car, car.routes[2].city), 10000)
            setTimeout(() => deliveryTimer(car), 1000)
          } else {
            car.working = false
          }
        }
      }
    } else {
      finance.value.fuelCost += 200
      countMoney()
      setTimeout(() => deliveryTimer(car), 900)
    }
  } else {
    setTimeout(() => deliveryTimer(car), 900)
  }
}


const countMoney = () => {
  finance.value.expenses = finance.value.fuelCost + finance.value.serviceCost + finance.value.repairCost + finance.value.salaryCost
  finance.value.profit = finance.value.income - finance.value.expenses
  if (finance.value.profit > 0) {
    finance.value.tax = finance.value.profit * 0.2
  } else {
    finance.value.tax = 0
  }
  finance.value.netProfit = finance.value.profit - finance.value.tax
}


const delivery = (car, city) => {
  if (car.broken) {
    setTimeout(() => delivery(car, city), 1000)
  } else {
    if (car.items.findIndex((item) => item.destination === city) !== -1) {
      finance.value.income += car.items[car.items.findIndex((item) => item.destination === city)].cost
      countMoney()
      orderHistory.value.push({
        cargo: car.items[car.items.findIndex((item) => item.destination === city)].title,
        car: car.name,
        route: car.items[car.items.findIndex((item) => item.destination === city)].departure + "-" + car.items[car.items.findIndex((item) => item.destination === city)].destination,
        weight: car.items[car.items.findIndex((item) => item.destination === city)].weight,
        volume: car.items[car.items.findIndex((item) => item.destination === city)].volume,
        cost: car.items[car.items.findIndex((item) => item.destination === city)].cost
      })
      car.routes[car.routes.findIndex((route) => route.city === city)].cargos--
      car.items.splice(car.items.findIndex((item) => item.destination === city), 1)
      setTimeout(() => delivery(car, city), 1000)
    } else {
      finance.value.salaryCost += 2000
      countMoney()
      car.delivering = false
    }
  }
}

const progressTimer = (car, time, maxTime) => {
  if (time !== 0 && !car.broken) {
    time--
    car.time = time
    document.getElementById(car.name).value = 100 - Math.floor(time * 100 / maxTime)
    setTimeout(() => progressTimer(car, time, maxTime), 1000)
  }
}

const countCars = () => {
  for (let i in cars.value) {
    cars.value[i].usedVolume = 0
    cars.value[i].usedWeight = 0
    for (let j in cars.value[i].items) {
      cars.value[i].usedVolume += cars.value[i].items[j].volume
      cars.value[i].usedWeight += cars.value[i].items[j].weight
    }
  }
}

const drag = (e, cargo, source) => {
  let allow = true
  for (let i in cars.value) {
    if (cars.value[i].name === source) {
      if (cars.value[i].working) {
        alert("Машина находится в поездке")
        allow = false
      }
    }
  }
  e.dataTransfer.setData("source", source)
  e.dataTransfer.setData("cargo", JSON.stringify(cargo))
}

const drop = (e, car) => {
  let jsonCargo = JSON.parse(e.dataTransfer.getData("cargo"))
  let source = e.dataTransfer.getData("source")
  let allow = true
  if (car !== null) {
    for (let i in cars.value) {
      if (cars.value[i].name === car.name) {
        if (car.working) {
          alert("Машина находится в поездке")
          allow = false
        }
      }
    }
  }
  if (allow) {
    if (car !== null) {
      if (car.usedWeight + jsonCargo.weight > car.weight || car.usedVolume + jsonCargo.volume > car.volume) {
        alert("Превышение объема/веса")
      } else {
        let newItems = []
        for (let i in cars.value) {
          if (cars.value[i].name === source) {
            for (let j in cars.value[i].items) {
              if (cars.value[i].items[j].title !== jsonCargo.title) {
                newItems.push(cars.value[i].items[j])
              }
            }
            cars.value[i].items = newItems
          }
        }
        car.items.push(jsonCargo)
        let newCargos = []
        for (let i in cargos.value) {
          if (cargos.value[i].title !== jsonCargo.title) {
            newCargos.push(cargos.value[i])
          }
        }
        cargos.value = newCargos
      }
    } else {
      cargos.value.push(jsonCargo)
      let newItems = []
      for (let i in cars.value) {
        if (cars.value[i].name === source) {
          for (let j in cars.value[i].items) {
            if (cars.value[i].items[j].title !== jsonCargo.title) {
              newItems.push(cars.value[i].items[j])
            }
          }
          cars.value[i].items = newItems
        }
      }
    }
    countCars()
  }
}

let brokenCar = null;

let lostCargos = []

const loseCargos = () => {
    lostCargos = cargos.value
    cargos.value = []
    alert("Поломка в системе. Список грузов потерян")
    document.querySelector('.fix-button').disabled = false
}

const findCargos = () => {
    for (let i in lostCargos) {
        cargos.value.push(lostCargos[i])
    }
    document.querySelector('.fix-button').disabled = true
}

const breakCar = () => {
  if (brokenCar === null) {
      brokenCar = Math.floor(Math.random() + 0.5)
      cars.value[brokenCar].broken = true
      document.querySelector('.repair-button').disabled = false
      alert("Машина " + cars.value[brokenCar].name + " сломалась")
  }
}

const fixCar = () => {
  cars.value[brokenCar].broken = false
  finance.value.repairCost += 20000
  countMoney()
  brokenCar = null
  document.querySelector('.repair-button').disabled = true
}

const workTimer = () => {
  sec.value++
  if (sec.value === 30) {
    breakCar()
  }
  if (sec.value === 50) {
      loseCargos()
  }
  if (sec.value === 60) {
    sec.value = 0
    min.value++
    if (min.value === 60) {
      min.value = 0
      hrs.value++
    }
  }
}

const createOrder = () => {
  orderTime++
  if (orderTime === 3) {
    orderTime = 0
    let shuffleRoutes = routes
        .sort(() => {return .4 - Math.random()})
        .slice(0, 2)
    let time = 0
    let cost = 0
    switch (shuffleRoutes[0]) {
      case "Москва":
        time = 8
        cost = Math.floor(Math.random() * (40-10) + 10) * 50 + 3000
        break
      case "Санкт-Петербург":
        time = 8
        cost = Math.floor(Math.random() * (40-10) + 10) * 50 + 4000
        break
      case "Казань":
        time = 8
        cost = Math.floor(Math.random() * (40-10) + 10) * 50 + 1000
        break
    }
    cargos.value.push({
      title: 'Груз №' + orderNumber,
      weight: Math.floor(Math.random() * (15 - 5) + 5),
      volume: Math.floor(Math.random() * (20 - 5) + 5),
      departure: "Уфа",
      destination: shuffleRoutes[0],
      cost: cost,
      time: time
    })
    orderNumber++
  }
}

const start = () => {
  working = true
  document.querySelector('.start-button').disabled = true
  document.querySelector('.stop-button').disabled = false
}

const stop = () => {
  working = false
  document.querySelector('.start-button').disabled = false
  document.querySelector('.stop-button').disabled = true
}

const initCharts = () => {
    setTimeout(() => {
      moneyChart = new Chart(document.querySelector(".money-chart canvas"), {
        type: "line",
        data: {
          labels: data.value.map(row => row.time),
          datasets: [
            {
              label: 'Доходы',
              data: data.value.map(row => row.income)
            },
            {
              label: 'Расходы',
              data: data.value.map(row => row.expenses)
            }
          ]
        }
      })
      profitChart = new Chart(document.querySelector(".profit-chart canvas"), {
        type: "line",
        data: {
          labels: data.value.map(row => row.time),
          datasets: [
            {
              label: 'Прибыль',
              data: data.value.map(row => row.profit)
            },
            {
              label: 'Чистая прибыль',
              data: data.value.map(row => row.netProfit)
            }
          ]
        }
      })
    }, 500)
}

initCharts()

const work = () => {
  if (working) {
    workTimer()
    moneyChart.data.labels.push(sec.value+min.value*60+hrs.value*60*60)
    moneyChart.data.datasets[0].data.push(finance.value.income)
    moneyChart.data.datasets[1].data.push(finance.value.expenses)
    moneyChart.update()
    profitChart.data.labels.push(sec.value+min.value*60+hrs.value*60*60)
    profitChart.data.datasets[0].data.push(finance.value.profit)
    profitChart.data.datasets[1].data.push(finance.value.netProfit)
    profitChart.update()
    createOrder()
  }
  setTimeout(() => {work()}, 1000)
}

const autoDelivery = () => {
  if (auto) {
    if (cargos.value.length !== 0) {
      if (!cars.value[0].working && (cars.value[0].usedWeight + cargos.value[0].weight <= cars.value[0].weight || cars.value[0].usedVolume + cargos.value[0].volume <= cars.value[0].volume)) {
        cars.value[0].items.push(cargos.value[0])
        cargos.value.splice(0, 1)
        countCars()
      } else {
        if (!cars.value[0].working) {
          prepareDelivery(cars.value[0])
        }
        if (!cars.value[1].working && (cars.value[1].usedWeight + cargos.value[0].weight <= cars.value[1].weight || cars.value[1].usedVolume + cargos.value[0].volume <= cars.value[1].volume)) {
          cars.value[1].items.push(cargos.value[0])
          cargos.value.splice(0, 1)
          countCars()
        } else {
          if (!cars.value[1].working) {
            prepareDelivery(cars.value[1])
          }
        }
      }
    }
  }
  countCars()
  setTimeout(() => autoDelivery(), 1000)
}
autoDelivery()
work()
</script>

<template>
  <div class="container">
    <div class="left">
      <div class="desc">
        <h2>Система доставки грузов</h2>
        <p>ПИ-424Б Скитяева Анастасия Николаевна</p>
      </div>
      <div class="control-panel">
        <h2>Панель управления</h2>
        <p>Время работы: {{hrs}} часов {{min}} минут {{sec}} секунд</p>
        <button class="control-button start-button" @click="start">Начать получать заказы</button>
        <button class="control-button stop-button" @click="stop" disabled>Перестать получать заказы</button>
        <button class="control-button repair-button" @click="fixCar" disabled>Починить машину</button>
        <button class="control-button fix-button" @click="findCargos" disabled>Устранить ошибку в системе</button>
        <div class="auto-button">
          <input type="checkbox" name="auto" id="auto" @change="() => {auto = !auto}"><label for="auto">Автоматический контроль</label>
        </div>
      </div>
    </div>
    <div class="right">
      <v-tabs v-model="tab">
        <v-tab value="two">Графики</v-tab>
        <v-tab value="one">Доставка</v-tab>
        <v-tab value="three">История</v-tab>
      </v-tabs>
      <v-window v-model="tab">
        <v-window-item value="two" class="charts">
          <div class="info-panel">
            <h3>Панель информации</h3>
            <ul>
              <li>Затраты на топливо: {{finance.fuelCost}} рублей</li>
              <li>Затраты на оплату труда: {{finance.salaryCost}} рублей</li>
              <li>Затраты на починку: {{finance.repairCost}} рублей</li>
              <li>Расходы: {{finance.expenses}} рублей</li>
              <li>Доходы: {{finance.income}} рублей</li>
              <li>Прибыль: {{finance.profit}} рублей</li>
              <li>Налоги: {{finance.tax}} рублей</li>
              <li>Чистая прибыль: {{finance.netProfit}} рублей</li>
            </ul>
          </div>
          <div class="charts-charts">
            <div class="money-chart">
              <canvas id="money-chart"></canvas>
            </div>
            <div class="profit-chart">
              <canvas id="profit-chart"></canvas>
            </div>
          </div>
        </v-window-item>
        <v-window-item value="one" class="delivery">
          <div class="cargos-box" @drop="drop($event, null)" @dragover.prevent @dragenter.prevent>
            <h3>Грузы</h3>
            <div class="cargos">
              <div class="cargo" v-for="cargo in cargos" draggable="true" @dragstart="drag($event, cargo, null)">
                <h4>{{cargo.title}}</h4>
                <ul>
                  <li>Объем: {{cargo.volume}} м3</li>
                  <li>Вес: {{cargo.weight}} т</li>
                  <li>Стоимость: {{cargo.cost}} рублей</li>
                  <li>Маршрут: {{cargo.departure}}-{{cargo.destination}}</li>
                </ul>
              </div>
            </div>
          </div>
          <div class="cars-box">
            <h3>Машины</h3>
            <div class="cars">
              <div class="car" v-for="car in cars" @drop="drop($event, car)" @dragover.prevent @dragenter.prevent>
                <h3>{{car.name}}</h3>
                <button class="send-button" @click="prepareDelivery(car)">Отправить доставлять грузы</button>
                <div class="progress-bar">
                  <progress value="0" max="100" :id="car.name"></progress>
                  Осталось {{car.time}} сек.
                </div>
                <ul>
                  <li>Вес: {{car.usedWeight}}/{{car.weight}} т</li>
                  <li>Объем: {{car.usedVolume}}/{{car.volume}} м3</li>
                  <li>Грузы:
                    <ul>
                      <li class="car-item" v-for="item in car.items" :id="item.title" draggable="true" @dragstart="drag($event, item, car.name)">
                        {{item.title}}. {{item.weight}} т, {{item.volume}} м3, {{item.cost}} рублей. {{item.departure}}-{{item.destination}}.
                      </li>
                    </ul>
                  </li>
                </ul>
              </div>
            </div>
          </div>
        </v-window-item>
        <v-window-item value="three" class="history">
          <table>
            <thead>
              <tr>
                <td>№</td>
                <td>Машина</td>
                <td>Маршрут</td>
                <td>Вес</td>
                <td>Объем</td>
                <td>Стоимость</td>
              </tr>
            </thead>
            <tbody>
              <tr v-for="order in orderHistory">
                <td>{{order.cargo}}</td>
                <td>{{order.car}}</td>
                <td>{{order.route}}</td>
                <td>{{order.weight}} т</td>
                <td>{{order.volume}} м3</td>
                <td>{{order.cost}} рублей</td>
              </tr>
            </tbody>
          </table>
        </v-window-item>
      </v-window>
    </div>
  </div>
</template>

<style scoped>
td {
  border: 1px solid black;
  padding: 5px;
}
table {
  width: 100%;
  border: 1px solid black;
  border-collapse: collapse;
}

.info-panel {
  height: 100%;
  width: 30vh;
}

.charts {
  display: flex;
  height: 90vh;
}

.charts-charts {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 80vw;
}

.money-chart, .profit-chart {
  width: 90vw;
  height: 40vh;
}

#money-chart, #profit-chart {
  width: 90vw;
  height: 30vh;
}

button {
  border: 1px solid black;
  padding: 5px;
}

button:hover {
  background-color: #33d0d0;
  transition: 0.5s;
}

button:active {
  background-color: darkslategrey;
}

.send-button {
  margin: 5px 0;
}

.control-button {
  margin: 5px;
}

.delivery {
  display: flex;
}

.car-item {
  border: 1px solid black;
  padding: 10px;
  margin-top: 10px;
}

.car {
  min-height: 90vh;
  max-height: 90vh;
}

.cargo {
  margin-top: 10px;
  border: 1px solid black;
  text-align: center;
  min-width: 200px;
}

.cargos-box {
  min-width: 220px;
}

.cargos {
  overflow-y: scroll;
  max-height: 90vh;
}

.cargos-box, .cars-box {
  margin-left: 10px;
}

.cars-box, cars {
  width: 100%;
}

.car {
  margin: 5px 5px 5px 0;
}

.car {
  width: 50%;
}

.cars, cargos {
  display: flex;
}

ul {
  list-style-type: none;
}

.container {
  display: flex;
  flex-direction: row;
}

.desc {
  padding: 10px;
  background-color: #33d0d0;
}

.control-panel {
  padding: 10px;
  display: flex;
  flex-direction: column;
}

.left {
  background-color: rgb(211, 209, 209);
  height: 100vh;
  min-width: 32vh;
  max-width: 32vh;
}

.right {
  /*noinspection CssInvalidPropertyValue*/
  width: -webkit-fill-available;
}

.history {
  height: 90vh;
  overflow-y: scroll;
}
</style>
