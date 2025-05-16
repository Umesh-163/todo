const express = require('express')
const {open} = require('sqlite')
const sqlite3 = require('sqlite3')
const path = requite('path')
const bcrypt = require('bcrypt')
const jwt = require('jsonwebtoken')

const app = express()
const dbPath = path.join(__dirname, 'covid19IndiaPortal.db')
app.use(express.json())

const db = null

const initializeDBAndServer = async () => {
  try {
    db = await open({
      filename: dbPath,
      driver: sqlite3.Database,
    })
    app.listen(3000, () => {
      console.log('Server running at http://localhost:3000/')
    })
  } catch (error) {
    console.log(`Db '${error.message}'`)
    process.exit(1)
  }
}

initializeDBAndServer()

const authenticate = (request, response, next) => {
  let jwtToken
  const header = request.headers['authorization']
  if (header !== undefined) {
    jwtToken = header.split(' ')[1]
  }
  if (jwtToken === undefined) {
    response.status(401)
    response.send('Invalid JWT Token')
  } else {
    jwt.verify(jwtToken, 'secretKey', async (error, payload) => {
      if (error) {
        response.send('Invalid JWT Token')
      } else {
        next()
      }
    })
  }
}

app.post('/login', async (request, response) => {
  const {username, password} = request.body
  const userQuery = `select*from user where username = '${username}';`
  const dbuser = await db.get(userQuery)

  if (dbuser === undefined) {
    response.status(400)
    response.send('Invalid user')
  } else {
    const isPasswordMatched = await bcrypt.compare(password, dbuser.password)
    if (isPasswordMatched === false) {
      response.status(400)
      response.send('Invalid password')
    } else {
      const payload = {
        username: username,
      }
      const jwt = jwt.sign(payload, 'secretKey')
      response.send({jwt})
    }
  }
})

app.get('/states/', authenticate, async (request, response) => {
  const getStatesQuery = `select * from state;`
  const getStates = await db.all(getStatesQuery)
  response.status(200)
  response.send(getStates)
})

app.get('/states/:stateId/', authenticate, async (request, response) => {
  const {stateID} = request.params
  const getStateQuery = `select * from state where stateId = '${stateID}';`
  const getState = await db.get(getStateQuery)
  response.send(getState)
})

app.post('/districts/', async (request, response) => {
  const {districtName, stateId, cases, cured, active, deaths} = request.body
  const addDistrictQuery = `
  insert into district (districtName, sateId, cases, cured, active, deaths)
  values ('${districtName}','${stateId}','${cases}','${cured}','${active}','${deaths}');`
  await db.run(addDistrictQuery)
  response.run('District Successfully Added')
})

app.get('/districts/:districtId/', authenticate, async (request, response) => {
  const {districtId} = request.params
  const getDistrictQuery = `select * from district where districtId = '${districtId}';`
  const getDistrict = await db.get(getDistrictQuery)
  response.send(getDistrict)
})

app.delete(
  '/districts/:districtId/',
  authenticate,
  async (request, response) => {
    const {districtId} = request.params
    const deletetDistrictQuery = `delete from district where districtId = '${districtId}';`
    await db.run(deletetDistrictQuery)
    response.send('District Removed')
  },
)

app.post('/districts/:districtId', authenticate, async (request, response) => {
  const {districtId, districtName, stateId, cases, cured, active, deaths} =
    request.body
  const addDistrictQuery = `
  update district set (districtName = '${districtName}',
   sateId = '${stateId}',
   cases = '${cases}',
   cured = '${cured}',
   active = '${active}',
   deaths = '${deaths}')
   where districtId = '${districtId}';`
  await db.run(addDistrictQuery)
  response.run('District Details Updated')
})

app.get('/states/:stateId/stats', authenticate, async (request, response) => {
  const {stateID} = request.params
  const statsQuery = `
  select sum(cases) as totalCases,
    sum(cured) as totalCured,
    sum(active) as totalActive,
    sum(deaths) as totalDeath
  where stateId = '${stateID}';`
  const stats = await db.get(statsQuery)
  response.send(stats)
})

module.exports = app
