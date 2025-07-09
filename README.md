# TO-DO-List
Here I making T-DO List in python using GUI and this is create task and finish task multiple but here I like choose 3 task  such as Task1,Task2 and Task3 and finish task completely.
import json
import os
from datetime import datetime
from enum import Enum
import tkinter as tk
from tkinter import ttk, messagebox

class Priority(Enum):
    LOW = "Low"
    MEDIUM = "Medium"
    HIGH = "High"

class Task:
    def __init__(self, description: str, priority: Priority = Priority.MEDIUM):
        self.description = description
        self.priority = priority
        self.created_at = datetime.now()
        self.completed = False
        self.completed_at = None

    def complete(self):
        self.completed = True
        self.completed_at = datetime.now()

    def to_dict(self):
        return {
            "description": self.description,
            "priority": self.priority.value,
            "created_at": self.created_at.isoformat(),
            "completed": self.completed,
            "completed_at": self.completed_at.isoformat() if self.completed else None
        }

    @classmethod
    def from_dict(cls, data):
        task = cls(data["description"], Priority(data["priority"]))
        task.created_at = datetime.fromisoformat(data["created_at"])
        task.completed = data["completed"]
        if task.completed and data["completed_at"]:
            task.completed_at = datetime.fromisoformat(data["completed_at"])
        return task

class TodoList:
    def __init__(self, filename="tasks.json"):
        self.filename = filename
        self.tasks = []
        self.load_tasks()

    def add_task(self, description: str, priority: Priority):
        task = Task(description, priority)
        self.tasks.append(task)
        self.save_tasks()
        return task

    def get_tasks(self, include_completed=False):
        if include_completed:
            return self.tasks
        return [task for task in self.tasks if not task.completed]

    def complete_task(self, task_index):
        if 0 <= task_index < len(self.tasks):
            self.tasks[task_index].complete()
            self.save_tasks()

    def remove_task(self, task_index):
        if 0 <= task_index < len(self.tasks):
            self.tasks.pop(task_index)
            self.save_tasks()

    def clear_completed(self):
        initial_count = len(self.tasks)
        self.tasks = [task for task in self.tasks if not task.completed]
        removed_count = initial_count - len(self.tasks)
        self.save_tasks()
        return removed_count

    def save_tasks(self):
        with open(self.filename, "w") as f:
            json.dump([task.to_dict() for task in self.tasks], f)

    def load_tasks(self):
        if os.path.exists(self.filename):
            with open(self.filename, "r") as f:
                try:
                    data = json.load(f)
                    self.tasks = [Task.from_dict(task_data) for task_data in data]
                except json.JSONDecodeError:
                    self.tasks = []

class TodoGUI(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Productivity Pulse - To-Do List")
        self.geometry("600x400")
        self.todo = TodoList()
        
        self.create_widgets()
        self.refresh_list()
    
    def create_widgets(self):
        # Input Frame
        input_frame = ttk.Frame(self)
        input_frame.pack(pady=10, padx=10, fill=tk.X)
        
        # Task Entry
        self.task_entry = ttk.Entry(input_frame, width=40)
        self.task_entry.pack(side=tk.LEFT, padx=5)
        
        # Priority Combobox
        self.priority_var = tk.StringVar()
        self.priority_combo = ttk.Combobox(
            input_frame, 
            textvariable=self.priority_var,
            values=[p.value for p in Priority],
            state="readonly",
            width=10
        )
        self.priority_combo.current(1)  # Default to Medium
        self.priority_combo.pack(side=tk.LEFT, padx=5)
        
        # Add Button
        add_btn = ttk.Button(input_frame, text="Add Task", command=self.add_task)
        add_btn.pack(side=tk.LEFT, padx=5)
        
        # List Control Frame
        ctrl_frame = ttk.Frame(self)
        ctrl_frame.pack(fill=tk.X, padx=10)
        
        show_completed_lbl = ttk.Label(ctrl_frame, text="Show:")
        show_completed_lbl.pack(side=tk.LEFT, padx=5)
        
        self.show_completed = tk.BooleanVar()
        show_completed_btn = ttk.Checkbutton(
            ctrl_frame,
            variable=self.show_completed,
            command=self.refresh_list
        )
        show_completed_btn.pack(side=tk.LEFT)
        
        clear_btn = ttk.Button(ctrl_frame, text="Clear Completed", command=self.clear_completed)
        clear_btn.pack(side=tk.RIGHT, padx=5)
        
        # Task List
        self.tree = ttk.Treeview(
            self,
            columns=("priority", "description", "created"),
            show="headings",
            height=15
        )
        
        self.tree.heading("priority", text="Priority")
        self.tree.heading("description", text="Description")
        self.tree.heading("created", text="Created")
        self.tree.column("priority", width=80)
        self.tree.column("description", width=300)
        self.tree.column("created", width=150)
        
        self.tree.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
        
        # Context Menu
        self.context_menu = tk.Menu(self, tearoff=0)
        self.context_menu.add_command(label="Complete", command=self.complete_selected)
        self.context_menu.add_command(label="Delete", command=self.delete_selected)
        
        self.tree.bind("<Button-3>", self.show_context_menu)
    
    def add_task(self):
        description = self.task_entry.get().strip()
        if not description:
            messagebox.showwarning("Warning", "Task description cannot be empty!")
            return
            
        try:
            priority = Priority(self.priority_var.get())
        except ValueError:
            priority = Priority.MEDIUM
            
        self.todo.add_task(description, priority)
        self.task_entry.delete(0, tk.END)
        self.refresh_list()
    
    def refresh_list(self):
        for item in self.tree.get_children():
            self.tree.delete(item)
            
        tasks = self.todo.get_tasks(include_completed=self.show_completed.get())
        
        for i, task in enumerate(tasks):
            status = "✔ " if task.completed else ""
            self.tree.insert(
                "", 
                tk.END, 
                values=(
                    task.priority.value,
                    f"{status}{task.description}",
                    task.created_at.strftime("%Y-%m-%d %H:%M")
                ),
                tags=("completed" if task.completed else "")
            )
        
        self.tree.tag_configure("completed", foreground="gray")
    
    def show_context_menu(self, event):
        item = self.tree.identify_row(event.y)
        if item:
            self.tree.selection_set(item)
            self.context_menu.post(event.x_root, event.y_root)
    
    def complete_selected(self):
        selected = self.tree.selection()
        if selected:
            index = self.tree.index(selected[0])
            self.todo.complete_task(index)
            self.refresh_list()
    
    def delete_selected(self):
        selected = self.tree.selection()
        if selected:
            index = self.tree.index(selected[0])
            self.todo.remove_task(index)
            self.refresh_list()
    
    def clear_completed(self):
        count = self.todo.clear_completed()
        messagebox.showinfo("Info", f"Cleared {count} completed tasks")
        self.refresh_list()

def run_cli():
    todo = TodoList()
    print("\nWelcome to Productivity Pulse - Command Line To-Do List\n")
    print("Available commands: add, list, list-all, complete, delete, clear, exit")
    
    while True:
        try:
            command = input("\n> ").strip().split()
            if not command:
                continue
                
            cmd = command[0].lower()
            
            if cmd == "add" and len(command) >= 2:
                priority = Priority.MEDIUM
                if len(command) >= 3:
                    try:
                        priority = Priority[command[-1].upper()]
                        description = " ".join(command[1:-1])
                    except KeyError:
                        description = " ".join(command[1:])
                else:
                    description = " ".join(command[1:])
                
                todo.add_task(description, priority)
                print(f"Added: {description} ({priority.value})")
                
            elif cmd == "list":
                tasks = todo.get_tasks()
                if tasks:
                    print("\nActive Tasks:")
                    for i, task in enumerate(tasks, 1):
                        print(f"{i}. {task.priority.value}: {task.description}")
                else:
                    print("\nNo active tasks!")
                    
            elif cmd == "list-all":
                tasks = todo.get_tasks(include_completed=True)
                if tasks:
                    print("\nAll Tasks:")
                    for i, task in enumerate(tasks, 1):
                        status = "[✓]" if task.completed else "[ ]"
                        print(f"{i}. {status} {task.priority.value}: {task.description}")
                else:
                    print("\nNo tasks!")
                    
            elif cmd in ("complete", "delete") and len(command) == 2:
                try:
                    index = int(command[1]) - 1
                    if 0 <= index < len(todo.tasks):
                        if cmd == "complete":
                            todo.tasks[index].complete()
                            print(f"Completed: {todo.tasks[index].description}")
                        else:
                            desc = todo.tasks[index].description
                            todo.remove_task(index)
                            print(f"Deleted: {desc}")
                        todo.save_tasks()
                    else:
                        print("Invalid task number!")
                except ValueError:
                    print("Please enter a valid task number!")
                    
            elif cmd == "clear":
                count = todo.clear_completed()
                print(f"Cleared {count} completed tasks")
                
            elif cmd == "exit":
                print("\nGoodbye! Your tasks have been saved.\n")
                break
                
            else:
                print("Invalid command. Try: add, list, list-all, complete, delete, clear, exit")
                
        except KeyboardInterrupt:
            print("\nGoodbye! Your tasks have been saved.\n")
            break
        except Exception as e:
            print(f"Error: {e}")

if __name__ == "__main__":
    print("Choose interface mode:")
    print("1. Command Line Interface (CLI)")
    print("2. Graphical User Interface (GUI)")
    
    while True:
        choice = input("> ").strip()
        if choice == "1":
            run_cli()
            break
        elif choice == "2":
            app = TodoGUI()
            app.mainloop()
            break
        else:
            print("Please enter 1 for CLI or 2 for GUI")
![Screenshot 2025-07-09 104942](https://github.com/user-attachments/assets/d1eb91de-cfad-4a91-b6f6-f901e81266c1)


