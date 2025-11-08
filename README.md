# desafio-fullstack-veritas
Este desafio é uma oportunidade excelente para demonstrar minhas habilidades em construir uma aplicação Fullstack moderna e funcional, integrando duas tecnologias de alto valor no mercado: React (frontend) e Go (backend).  Meu objetivo não é apenas entregar um código que funcione, mas sim apresentar uma solução que reflita boas práticas.

# Backend (Go) / net/http e sync.RWMutex - Uso da biblioteca padrão do Go para manter o projeto leve. O sync.RWMutex foi implementado no acesso ao mapa de tarefas para garantir thread safety no armazenamento em memória, prevenindo race conditions.
package main

// Task representa uma única tarefa no Kanban
type Task struct {
	ID          string `json:"id"`
	Title       string `json:"title"`
	Description string `json:"description,omitempty"`
	Status      string `json:"status"` // A Fazer, Em Progresso, Concluídas
}

// Constantes para os status do Kanban
const (
	StatusTodo       = "A Fazer"
	StatusInProgress = "Em Progresso"
	StatusDone       = "Concluídas"
)

// Estrutura para receber atualizações parciais
type TaskUpdate struct {
	Title       *string `json:"title,omitempty"`
	Description *string `json:"description,omitempty"`
	Status      *string `json:"status,omitempty"`
}

# Backend (Go) / Armazenamento em map[string]Task - O mapa permite a busca e atualização rápidas (tempo constante $O(1)$) de tarefas usando o ID como chave, o que é ideal para o CRUD.
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strings"
	"sync"

	"github.com/google/uuid"
)

// tasksDB é o armazenamento em memória
var tasksDB = make(map[string]Task)
var mutex = &sync.RWMutex{} // Mutex para garantir segurança em concorrência

// initData popula o DB com algumas tarefas iniciais
func initData() {
	mutex.Lock()
	defer mutex.Unlock()

	tasksDB[uuid.New().String()] = Task{ID: uuid.New().String(), Title: "Configurar Backend Go", Status: StatusDone}
	tasksDB[uuid.New().String()] = Task{ID: uuid.New().String(), Title: "Criar Estrutura React", Status: StatusInProgress}
	tasksDB[uuid.New().String()] = Task{ID: uuid.New().String(), Title: "Documentação User Flow", Status: StatusTodo}
}

// setupCORSHandler lida com as regras de CORS
func setupCORSHandler(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", "*") 
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type")

		if r.Method == "OPTIONS" {
			w.WriteHeader(http.StatusOK)
			return
		}
		handler.ServeHTTP(w, r)
	})
}

func writeJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

func httpError(w http.ResponseWriter, status int, message string) {
	writeJSON(w, status, map[string]string{"error": message})
}

// TaskHandler lida com requisições /tasks e /tasks/{id}
func TaskHandler(w http.ResponseWriter, r *http.Request) {
	parts := strings.Split(r.URL.Path, "/")
	taskID := ""
	if len(parts) > 2 && parts[2] != "" {
		taskID = parts[2]
	}

	switch r.Method {
	case "GET":
		if taskID == "" {
			getTasks(w, r)
		} else {
			getTaskByID(w, r, taskID)
		}
	case "POST":
		createTask(w, r)
	case "PUT":
		if taskID != "" {
			updateTask(w, r, taskID)
		} else {
			httpError(w, http.StatusMethodNotAllowed, "PUT method requires an ID")
		}
	case "DELETE":
		if taskID != "" {
			deleteTask(w, r, taskID)
		} else {
			httpError(w, http.StatusMethodNotAllowed, "DELETE method requires an ID")
		}
	default:
		httpError(w, http.StatusMethodNotAllowed, "Method not supported")
	}
}

func getTasks(w http.ResponseWriter, r *http.Request) {
	mutex.RLock()
	defer mutex.RUnlock()

	tasksList := []Task{}
	for _, task := range tasksDB {
		tasksList = append(tasksList, task)
	}
	writeJSON(w, http.StatusOK, tasksList)
}

func getTaskByID(w http.ResponseWriter, r *http.Request, id string) {
	mutex.RLock()
	defer mutex.RUnlock()

	task, ok := tasksDB[id]
	if !ok {
		httpError(w, http.StatusNotFound, "Task not found")
		return
	}
	writeJSON(w, http.StatusOK, task)
}

func createTask(w http.ResponseWriter, r *http.Request) {
	var newTask Task
	if err := json.NewDecoder(r.Body).Decode(&newTask); err != nil {
		httpError(w, http.StatusBadRequest, "Invalid request payload")
		return
	}

	// Validação (Título obrigatório e Status inicial)
	if strings.TrimSpace(newTask.Title) == "" {
		httpError(w, http.StatusBadRequest, "Title is mandatory")
		return
	}
	
	// Define ID e Status inicial se não vier no payload
	newTask.ID = uuid.New().String()
	if newTask.Status == "" {
		newTask.Status = StatusTodo
	}

	mutex.Lock()
	tasksDB[newTask.ID] = newTask
	mutex.Unlock()

	writeJSON(w, http.StatusCreated, newTask)
}

func updateTask(w http.ResponseWriter, r *http.Request, id string) {
	var updateData TaskUpdate
	
	body, err := io.ReadAll(r.Body)
	if err != nil {
		httpError(w, http.StatusInternalServerError, "Failed to read request body")
		return
	}
	
	// Tenta decodificar o corpo da requisição na estrutura de atualização
	if err := json.Unmarshal(body, &updateData); err != nil {
		httpError(w, http.StatusBadRequest, "Invalid request payload")
		return
	}

	mutex.Lock()
	defer mutex.Unlock()

	task, ok := tasksDB[id]
	if !ok {
		httpError(w, http.StatusNotFound, "Task not found")
		return
	}

	// Aplica atualizações se existirem
	if updateData.Title != nil {
		if strings.TrimSpace(*updateData.Title) == "" {
			httpError(w, http.StatusBadRequest, "Title cannot be empty")
			return
		}
		task.Title = *updateData.Title
	}
	if updateData.Description != nil {
		task.Description = *updateData.Description
	}
	if updateData.Status != nil {
		// Validação de status
		validStatus := (*updateData.Status == StatusTodo || *updateData.Status == StatusInProgress || *updateData.Status == StatusDone)
		if !validStatus {
			httpError(w, http.StatusBadRequest, "Invalid status value")
			return
		}
		task.Status = *updateData.Status
	}

	tasksDB[id] = task // Salva a tarefa atualizada
	writeJSON(w, http.StatusOK, task)
}

func deleteTask(w http.ResponseWriter, r *http.Request, id string) {
	mutex.Lock()
	defer mutex.Unlock()

	if _, ok := tasksDB[id]; !ok {
		httpError(w, http.StatusNotFound, "Task not found")
		return
	}

	delete(tasksDB, id)
	w.WriteHeader(http.StatusNoContent)
}
# Backend (Go) / Structs Separadas para PUT	A struct TaskUpdate foi criada para o método PUT. Isso permite receber apenas os campos que o usuário deseja atualizar (ex: Status para mover ou Title para editar), facilitando a lógica de atualização parcial.
package main

import (
	"log"
	"net/http"
)

func main() {
	// Inicializa alguns dados de exemplo
	initData()
	
	// Cria um novo mux (roteador)
	mux := http.NewServeMux()
	
	// Rota para manipular /tasks e /tasks/{id}
	mux.HandleFunc("/tasks/", TaskHandler)

	// Aplica o middleware CORS
	handler := setupCORSHandler(mux)

	log.Println("Server running on port 8080...")
	// Loga erros de inicialização do servidor
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}
# Frontend (React)	Gerenciamento de Estado Local	O estado (tasks, isLoading, error) é gerenciado no componente principal (App.jsx) usando React Hooks (useState, useEffect). Isso atende ao escopo mínimo de uma aplicação simples, mantendo a arquitetura de dados linear.
import React, { useState, useEffect } from 'react';
import './App.css'; 

// URL base da API
const API_URL = 'http://localhost:8080/tasks'; 

// Componente para um item de tarefa
const TaskItem = ({ task, onEdit, onDelete, onMove }) => {
    // Estado para o modo de edição
    const [isEditing, setIsEditing] = useState(false);
    const [title, setTitle] = useState(task.Title);
    const [description, setDescription] = useState(task.Description);

    const handleSave = () => {
        if (title.trim() === '') return;
        onEdit(task.ID, title, description);
        setIsEditing(false);
    };

    return (
        <div className="task-item">
            {isEditing ? (
                <div className="edit-form">
                    <input 
                        value={title} 
                        onChange={(e) => setTitle(e.target.value)} 
                        placeholder="Título" 
                    />
                    <textarea 
                        value={description} 
                        onChange={(e) => setDescription(e.target.value)} 
                        placeholder="Descrição (Opcional)"
                    />
                    <button onClick={handleSave}>Salvar</button>
                    <button onClick={() => setIsEditing(false)}>Cancelar</button>
                </div>
            ) : (
                <>
                    <h4>{task.Title}</h4>
                    <p>{task.Description}</p>
                    <div className="task-actions">
                        <button onClick={() => setIsEditing(true)}>Editar</button>
                        <button onClick={() => onDelete(task.ID)}>Excluir</button>
                    </div>

                    <div className="move-actions">
                        {task.Status !== 'A Fazer' && (
                            <button onClick={() => onMove(task.ID, 'A Fazer')}>{'<-'}</button>
                        )}
                        {task.Status !== 'Concluídas' && (
                            <button onClick={() => onMove(task.ID, task.Status === 'A Fazer' ? 'Em Progresso' : 'Concluídas')}>
                                {task.Status === 'A Fazer' ? 'Em Progresso ->' : 'Concluídas ->'}
                            </button>
                        )}
                    </div>
                </>
            )}
        </div>
    );
};

// Componente para a coluna Kanban
const KanbanColumn = ({ status, tasks, onEdit, onDelete, onMove }) => {
    return (
        <div className="kanban-column">
            <h3>{status} ({tasks.length})</h3>
            {tasks.map(task => (
                <TaskItem 
                    key={task.ID} 
                    task={task} 
                    onEdit={onEdit} 
                    onDelete={onDelete} 
                    onMove={onMove}
                />
            ))}
        </div>
    );
};

// Componente principal
function App() {
    const [tasks, setTasks] = useState([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState(null);

    // Formulário de Nova Tarefa
    const [newTaskTitle, setNewTaskTitle] = useState('');
    const [newTaskDescription, setNewTaskDescription] = useState('');

    // --- API HANDLERS ---

    // 1. GET: Buscar todas as tarefas
    const fetchTasks = async () => {
        setIsLoading(true);
        setError(null);
        try {
            const response = await fetch(API_URL);
            if (!response.ok) {
                throw new Error(`Erro: ${response.status}`);
            }
            const data = await response.json();
            setTasks(data);
        } catch (err) {
            setError('Falha ao carregar tarefas.');
            console.error(err);
        } finally {
            setIsLoading(false);
        }
    };

    useEffect(() => {
        fetchTasks();
    }, []);

    // 2. POST: Criar nova tarefa
    const handleAddTask = async (e) => {
        e.preventDefault();
        if (newTaskTitle.trim() === '') return;
        
        const newTask = {
            Title: newTaskTitle,
            Description: newTaskDescription,
            Status: 'A Fazer', // Status inicial
        };

        try {
            const response = await fetch(API_URL, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(newTask),
            });

            if (!response.ok) {
                throw new Error('Falha ao adicionar tarefa.');
            }
            const addedTask = await response.json();
            setTasks([...tasks, addedTask]);
            setNewTaskTitle('');
            setNewTaskDescription('');
        } catch (err) {
            setError('Erro ao criar tarefa.');
            console.error(err);
        }
    };
    
    // 3. PUT: Mover/Atualizar tarefa
    const handleUpdateTask = async (id, updatedFields) => {
        try {
            const response = await fetch(`${API_URL}${id}`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(updatedFields),
            });

            if (!response.ok) {
                throw new Error('Falha ao atualizar tarefa.');
            }
            const updatedTask = await response.json();
            
            // Atualiza o estado local
            setTasks(tasks.map(t => (t.ID === id ? updatedTask : t)));

        } catch (err) {
            setError('Erro ao atualizar tarefa.');
            console.error(err);
        }
    };

    // Função auxiliar para mover (chama o PUT)
    const handleMoveTask = (id, newStatus) => {
        handleUpdateTask(id, { Status: newStatus });
    };

    // Função auxiliar para editar (chama o PUT)
    const handleEditTask = (id, newTitle, newDescription) => {
        handleUpdateTask(id, { Title: newTitle, Description: newDescription });
    };

    // 4. DELETE: Excluir tarefa
    const handleDeleteTask = async (id) => {
        try {
            const response = await fetch(`${API_URL}${id}`, {
                method: 'DELETE',
            });

            if (!response.ok) {
                throw new Error('Falha ao excluir tarefa.');
            }
            
            // Atualiza o estado local (remove a tarefa)
            setTasks(tasks.filter(t => t.ID !== id));
        } catch (err) {
            setError('Erro ao excluir tarefa.');
            console.error(err);
        }
    };


    // Filtra tarefas por coluna
    const todoTasks = tasks.filter(t => t.Status === 'A Fazer');
    const inProgressTasks = tasks.filter(t => t.Status === 'Em Progresso');
    const doneTasks = tasks.filter(t => t.Status === 'Concluídas');

    return (
        <div className="app-container">
            <h1>Mini Kanban de Tarefas</h1>
            
            {error && <div className="feedback error">{error}</div>}

            <form className="add-task-form" onSubmit={handleAddTask}>
                <input 
                    value={newTaskTitle}
                    onChange={(e) => setNewTaskTitle(e.target.value)}
                    placeholder="Título da Nova Tarefa (Obrigatório)"
                    required
                />
                <input 
                    value={newTaskDescription}
                    onChange={(e) => setNewTaskDescription(e.target.value)}
                    placeholder="Descrição (Opcional)"
                />
                <button type="submit">Adicionar Tarefa</button>
            </form>

            {isLoading ? (
                <div className="feedback loading">Carregando tarefas...</div>
            ) : (
                <div className="kanban-board">
                    <KanbanColumn 
                        status="A Fazer" 
                        tasks={todoTasks} 
                        onEdit={handleEditTask}
                        onDelete={handleDeleteTask}
                        onMove={handleMoveTask}
                    />
                    <KanbanColumn 
                        status="Em Progresso" 
                        tasks={inProgressTasks} 
                        onEdit={handleEditTask}
                        onDelete={handleDeleteTask}
                        onMove={handleMoveTask}
                    />
                    <KanbanColumn 
                        status="Concluídas" 
                        tasks={doneTasks} 
                        onEdit={handleEditTask}
                        onDelete={handleDeleteTask}
                        onMove={handleMoveTask}
                    />
                </div>
            )}
        </div>
    );
}

export default App;

# Frontend (React)	Componentes Reutilizáveis	A interface foi dividida em KanbanColumn e TaskItem. Essa modularização melhora a clareza do código e facilita a manutenção, alinhando-se às boas práticas do React.
/* Estilos Gerais */
.app-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    font-family: Arial, sans-serif;
}

h1 {
    text-align: center;
    color: #333;
}

/* Feedback Visual */
.feedback {
    padding: 10px;
    margin-bottom: 15px;
    border-radius: 4px;
    font-weight: bold;
}
.loading {
    background-color: #e0f7fa;
    color: #00796b;
}
.error {
    background-color: #ffebee;
    color: #c62828;
    border: 1px solid #c62828;
}

/* Formulário de Adição */
.add-task-form {
    display: flex;
    gap: 10px;
    margin-bottom: 30px;
    padding: 15px;
    border: 1px solid #ccc;
    border-radius: 8px;
    background-color: #f9f9f9;
}
.add-task-form input, .add-task-form button {
    padding: 10px;
    border-radius: 4px;
    border: 1px solid #ddd;
}
.add-task-form button {
    background-color: #4CAF50;
    color: white;
    cursor: pointer;
    flex-shrink: 0; 
}
.add-task-form input:first-child {
    flex-grow: 1;
}

/* Kanban Board e Colunas */
.kanban-board {
    display: flex;
    gap: 20px;
}

.kanban-column {
    flex: 1;
    background-color: #f4f5f7;
    padding: 15px;
    border-radius: 8px;
    min-height: 400px;
}

.kanban-column h3 {
    text-align: center;
    margin-top: 0;
    padding-bottom: 10px;
    border-bottom: 2px solid #ddd;
}

/* Item de Tarefa */
.task-item {
    background-color: white;
    border: 1px solid #e0e0e0;
    border-radius: 4px;
    padding: 10px;
    margin-bottom: 10px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.05);
}

.task-item h4 {
    margin: 0 0 5px 0;
    color: #333;
}
.task-item p {
    margin: 0 0 10px 0;
    font-size: 0.9em;
    color: #666;
}

/* Ações */
.task-actions button, .move-actions button {
    padding: 5px 10px;
    margin-right: 5px;
    cursor: pointer;
    border: none;
    border-radius: 4px;
}

.task-actions button:first-child { background-color: #ff9800; color: white; }
.task-actions button:last-child { background-color: #f44336; color: white; }

.move-actions {
    margin-top: 10px;
    padding-top: 5px;
    border-top: 1px dashed #eee;
}
.move-actions button {
    background-color: #2196F3;
    color: white;
    font-size: 0.7em;
}

/* Formulário de Edição */
.edit-form input, .edit-form textarea {
    width: 95%;
    margin-bottom: 5px;
    padding: 5px;
}
.edit-form button {
    margin-top: 5px;
}
