<script>
    import { onMount } from "svelte";
    import { db } from "../../lib/firebase/firebase";
    import { authHandlers, authStore } from "../../store/store";
    import { doc, setDoc, getDoc } from "firebase/firestore";

    let todoList = [];
    let currTodo = "";
    let error = false;

    let inProgress = [];
    let doneList = [];

    // To track the currently dragged task and its source list
    let draggedTask = null;
    let draggedFrom = null;

    // Subscribe to authStore to ensure user is available
    authStore.subscribe((curr) => {
        if (curr.user) {
            loadTodos(); // Load tasks on user login
        }
    });

    function addTodo() {
        error = false;
        if (!currTodo) {
            error = true;
            return;
        }
        todoList = [...todoList, currTodo];
        currTodo = "";
        saveAndSyncTodos(); // Save changes
    }

    async function saveTodos() {
        try {
            const userRef = doc(db, "users", $authStore.user.uid);
            await setDoc(
                userRef,
                {
                    todos: todoList,
                    inProgress,
                    done: doneList,
                },
                { merge: true }
            );
        } catch (err) {
            console.error("There was an error saving your information", err);
        }
    }

    function saveToLocalStorage() {
        const data = {
            todos: todoList,
            inProgress,
            doneList,
        };
        localStorage.setItem("taskData", JSON.stringify(data));
    }

    function saveAndSyncTodos() {
        saveTodos(); // Save to Firebase
        saveToLocalStorage(); // Save to localStorage as fallback
    }

    async function loadTodos() {
        try {
            const userRef = doc(db, "users", $authStore.user.uid);
            const docSnap = await getDoc(userRef);

            if (docSnap.exists()) {
                const data = docSnap.data();
                todoList = data.todos || [];
                inProgress = data.inProgress || [];
                doneList = data.done || [];
            } else {
                console.log("No data found in Firebase. Loading from localStorage...");
                loadFromLocalStorage();
            }
        } catch (err) {
            console.error("Error loading todos from Firebase:", err);
            loadFromLocalStorage(); // Fallback to localStorage in case of errors
        }
    }

    function loadFromLocalStorage() {
    const savedData = localStorage.getItem("taskData");
    if (savedData) {
        const parsedData = JSON.parse(savedData); // Parse the saved data
        todoList = parsedData.todos || [];       // Assign values to `todoList`
        inProgress = parsedData.inProgress || []; // Assign values to `inProgress`
        doneList = parsedData.doneList || [];    // Assign values to `doneList`
    }
}


    function handleDragStart(event, task, from) {
        event.dataTransfer.setData("task", task);
        draggedTask = task;
        draggedFrom = from;
    }

    function handleDragOver(event) {
        event.preventDefault();
    }

    function handleDrop(to) {
        if (!draggedTask || draggedFrom === null) return;

        // Remove the task from its original list
        if (draggedFrom === "todoList") {
            todoList = todoList.filter((task) => task !== draggedTask);
        } else if (draggedFrom === "inProgress") {
            inProgress = inProgress.filter((task) => task !== draggedTask);
        } else if (draggedFrom === "doneList") {
            doneList = doneList.filter((task) => task !== draggedTask);
        }

        // Add the task to the new target list
        if (to === "todoList") {
            todoList = [...todoList, draggedTask];
        } else if (to === "inProgress") {
            inProgress = [...inProgress, draggedTask];
        } else if (to === "doneList") {
            doneList = [...doneList, draggedTask];
        }

        // Save the updated lists
        saveAndSyncTodos();

        // Clear the drag state
        draggedTask = null;
        draggedFrom = null;
    }

    // Load todos on component mount
    onMount(() => {
        loadFromLocalStorage(); // Load offline data
        if ($authStore.user) {
            loadTodos(); // Load online data
        }
    });
</script>

<div class="mainContainer">
    <div class="headerContainer">
        <h1>Todo List</h1>
        <div class="headerBtns">
            <button on:click={saveTodos}>
                <i class="fa-regular fa-floppy-disk" />
                <p>Save</p>
            </button>
            <button on:click={authHandlers.logout}>
                <i class="fa-solid fa-right-from-bracket" />
                <p>Logout</p>
            </button>
        </div>
    </div>
    <main>
        <!-- Todo List Section -->
        <div class="todoList">
            <h2>Todo</h2>
            {#if todoList.length === 0}
                <p>No tasks to do!</p>
            {/if}
            {#each todoList as todo}
                <div
                    class="task"
                    draggable="true"
                    on:dragstart={(event) => handleDragStart(event, todo, "todoList")}
                >
                    {todo}
                </div>
            {/each}
        </div>

        <!-- In Progress and Done Columns -->
        <div class="additional-columns">
            <div
                class="column in-progress"
                on:dragover|preventDefault={(event) => handleDragOver(event)}
                on:drop={() => handleDrop("inProgress")}
            >
                <h2>In Progress</h2>
                {#if inProgress.length === 0}
                    <p>No tasks in progress yet.</p>
                {/if}
                {#each inProgress as task}
                    <div
                        class="task"
                        draggable="true"
                        on:dragstart={(event) => handleDragStart(event, task, "inProgress")}
                    >
                        {task}
                    </div>
                {/each}
            </div>
            <div
                class="column done"
                on:dragover|preventDefault={(event) => handleDragOver(event)}
                on:drop={() => handleDrop("doneList")}
            >
                <h2>Done</h2>
                {#if doneList.length === 0}
                    <p>No completed tasks yet.</p>
                {/if}
                {#each doneList as task}
                    <div
                        class="task"
                        draggable="true"
                        on:dragstart={(event) => handleDragStart(event, task, "doneList")}
                    >
                        {task}
                    </div>
                {/each}
            </div>
        </div>
    </main>

    <!-- Add Todo Section -->
    <div class={"enterTodo " + (error ? "errorBorder" : "")}>
        <input bind:value={currTodo} type="text" placeholder="Enter todo" />
        <button on:click={addTodo}>ADD NEW</button>
    </div>
</div>

<style>
    /* Layout and Design */
    .mainContainer {
        display: flex;
        flex-direction: column;
        min-height: 100vh;
        gap: 24px;
        padding: 24px;
        width: 100%;
        max-width: 1000px;
        margin: 0 auto;
    }

    .headerContainer {
        display: flex;
        align-items: center;
        justify-content: space-between;
    }

    .headerBtns {
        display: flex;
        align-items: center;
        gap: 14px;
    }

    .headerContainer button {
        background: #003c5b;
        color: white;
        padding: 10px 18px;
        border: none;
        border-radius: 4px;
        font-weight: 700;
        display: flex;
        align-items: center;
        gap: 10px;
        cursor: pointer;
    }

    .headerContainer button i {
        font-size: 1.1rem;
    }

    .headerContainer button:hover {
        opacity: 0.7;
    }

    main {
        display: flex;
        flex-direction: column;
        gap: 8px;
        flex: 1;
    }

    .todoList {
        margin-bottom: 24px;
        padding: 16px;
        background: transparent; /* Set transparent background for the list */
        border-radius: 8px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        color: white;
    }

    .todoList h2 {
        margin-top: 0;
    }

    .enterTodo {
        display: flex;
        align-items: stretch;
        border: 1px solid #0891b2;
        border-radius: 5px;
        overflow: hidden;
    }

    .errorBorder {
        border-color: coral !important;
    }

    .enterTodo input {
        background: transparent;
        border: none;
        padding: 14px;
        color: white;
        flex: 1;
    }

    .enterTodo input:focus {
        outline: none;
    }

    .enterTodo button {
        padding: 0 28px;
        background: #003c5b;
        border: none;
        color: cyan;
        font-weight: 600;
        cursor: pointer;
    }

    .enterTodo button:hover {
        background: transparent;
    }

    .additional-columns {
        display: flex;
        gap: 16px;
        margin-top: 24px;
    }

    .column {
        flex: 1;
        border: 1px solid #ccc;
        border-radius: 8px;
        padding: 16px;
        min-height: 200px;
        background-color: transparent; /* Set transparent background for the columns */
        overflow-y: auto;
        color: white;
    }

    .column h2 {
        margin-top: 0;
    }

    /* Updated for 'in-progress' and 'done' columns to have black backgrounds */
    .in-progress {
        background-color: #003d5b75;
    }

    .done {
        background-color: #003d5b75;
    }

    .task {
    padding: 10px;
    margin-bottom: 8px;
    background: transparent; /* Make the task background transparent */
    border-left: 4px solid #2e77c2; /* Add a left border with your desired color */
    border-radius: 0 4px 4px 0; /* Round the right corners for a smoother look */
    cursor: grab;
    transition: background-color 0.2s ease;
    color: white; /* Ensure text is visible on transparent background */
}

.task:hover {
    background-color: rgba(46, 119, 194, 0.1); /* Slightly highlight on hover */
}

.task:active {
    cursor: grabbing;
}
</style>
