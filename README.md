const express = require('express')
const {open} = require('sqlite')
const sqlite3 = require('sqlite3')
const path = require('path')


const dbpath = path.join(__dirname, 'todoApplication.db')


const app = express()
app.use(express.json())


let db = null


const initializeDBAndServer = async () => {
  try {
    db = await open({
      filename: dbpath,
      driver: sqlite3.Database,
    })
    app.listen(3000, () => {
      console.log(' Server runnong at http://localhost:3000/')
    })
  } catch (e) {
    console.log(`DB Error: ${e.message}`)
    process.exit(1)
  }
}


initializeDBAndServer()


app.get('/todos/', async (request, response) => {
  let data = null
  let getTodosQuery = ''
  const {search_q = '', priority, status} = request.query


  switch (true) {
    case hasProrityAndStatusProperties(request.query):
      getTodosQuery = `
        select * from todo 
        where title like '%${search_q}%' 
        and status = '${status}' and
        priority = '${priority}';`
      break


    case hasProrityPropertiy(request.query):
      getTodosQuery = `
        select * from todo 
        where title like '%${search_q}%' and
        priority = '${priority}';`
      break


    case hasStatusProperty(request.query):
      getTodosQuery = `
        select * from todo 
        where title like '%${search_q}%' 
        and status = '${status}';`
      break


    default:
      getTodosQuery = `
        select * from todo 
        where title like '%${search_q}%';
        `
  }


  data = await db.all(getTodosQuery)
  response.send(data)
})


app.get('/todos/:todoId/', async (request, response) => {
  const {todoId} = request.params
  const getTodoQuery = `
  select * from todo
  where todoId = '%${todoId}%'`
  const data = await db.get(getTodoQuery)
  response.send(data)
})


app.post('/todos/', async (request, response) => {
  const {id, todo, priority, status} = request.body
  const addTodoQuery = `
  Insert into todo (id, todo, priority, status)
  values ('${id}','${todo}','${priority}','${status}');`
  await db.run(addTodoQuery)
  response.send('Todo Successfully Added')
})


app.put('/todos/:todoId/', async (request, response) => {
  const {todoId} = request.params
  const requestBody = request.body
  let updateColumn = ''


  switch (true) {
    case requestBody.status !== undefined:
      updateColumn = 'Status'
      break


    case requestBody.priority !== undefined:
      updateColumn = 'Priority'
      break


    case requestBody.todo !== undefined:
      updateColumn = 'Todo'
      break
  }


  const prevTodoQuery = `select * from todo where todoId = '${todoId}';`
  const prevTodo = await db.get(prevTodoQuery)


  const {
    todo = prevTodo.tod,
    priority = prevTodo.priority,
    status = prevTodo.status,
  } = request.body
  const updateTodoQuery = `
    Update todo set 
    todo = '${todo}',
    priority = '${priority}',
    status = '${status}'
    where todoId = '${todoId}';`


  await db.run(updateTodoQuery)
  response.send(`'${updateColumn}' Updated`)
})


app.delete('/todos/:todoId/', async (request, response) => {
  const {todoId} = request.params
  const deleteTodo = `
  Delete from todo where todoId = '${todoId}';
  `
  await db.run(deleteTodo)
  response.send('Todo Deleted')
})


module.exports = app
