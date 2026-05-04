Ansible 的执行机制非常严谨，理解其**加载顺序**、**变量作用域**和**生命周期**对于编写复杂的 Playbook（如你提供的 `add-clickhouse.yml`）至关重要。

## 1. Ansible 的加载与执行顺序

Ansible 的处理流程可以分为两个主要阶段：**解析阶段 (Parsing)** 和 **执行阶段 (Execution)**。

### A. 解析阶段 (静态处理)

在真正连接任何主机之前，Ansible 会先读取所有文件：

1. **读取 Inventory**: 加载 `hosts_4`，解析主机、组、组关系 (`children`) 和初始变量。（**全局可访问**）
2. **读取 Playbook**: 解析 YAML 结构。
3. **处理 `import_*` 语句**:
   - 在你的文件中，`import_role: name: clickhouse` 是**静态导入**。
   - 导入路径：会依次检索，包括了当前文件同级目录、上级目录、项目根目录等。通过`ansible-config dump | grep DEFAULT_ROLES_PATH` 命令可查
   - Ansible 会立即读取 `roles/clickhouse/tasks/main.yml` 等文件，并将其任务"展开"插入到当前 Playbook 的任务列表中。
   - *注意*：此时 `when` 条件**不会**被评估，任务只是被加载进内存。
4. **处理 `include_*` 语句**:
   - 如果是 `include_role` 或 `include_tasks`，它们不会被立即展开，而是保留为一个动态包含指令，直到执行到该行时才处理。

### B. 执行阶段 (动态处理)

针对 `hosts: "{{ NODE_TO_ADD }}"` 中的每一台主机，按顺序执行：

1. **建立连接**: 根据 `ansible_connection` 等变量 SSH 连接到目标主机。
2. **Gathering Facts**:
   - 默认执行 `setup` 模块，收集系统信息（OS, IP, CPU 等），存入 `ansible_facts`。
   - *在你的文件中*：第一步任务是 `service_facts`，它会额外收集服务状态并更新 `ansible_facts['services']`。
3. **执行 Tasks**:
   - 按顺序执行任务列表。
   - 遇到 `when` 条件时，使用当前主机的变量和 facts 进行求值。如果为 False，跳过该任务。
   - 遇到 `delegate_to` 时，临时切换上下文到委托的主机执行命令，但变量引用通常仍基于原主机（除非显式指定）。

---

## 2. 变量作用域 (Scope) 与优先级

Ansible 变量的作用域决定了变量在哪里可见，而优先级决定了当变量名冲突时谁生效。

### A. 作用域范围

1. **Global (全局)**: 通过命令行 `-e` 传入的变量，或在配置文件中定义的变量。作用于所有 Play 和 Host。
2. **Play (剧本级)**: 在 Play 级别定义的 `vars:` 或 `vars_files:`。仅作用于当前 Play 中的所有 Host。
3. **Host (主机级)**:
   - Inventory 中定义的主机变量（如 `hosts_4` 中的 `NGINX_ROLE=MASTER`）。
   - `host_vars/` 目录下的文件。
   - 仅作用于特定主机。
4. **Group (组级)**:
   - Inventory 中 `[group:vars]` 定义的变量。
   - `group_vars/` 目录下的文件。
   - 作用于组内所有主机。子组变量优先于父组变量。
5. **Task/Role (任务/角色级)**:
   - 在 Role 的 `defaults/main.yml` 或 `vars/main.yml` 中定义。
   - 仅在 Role 或被包含的任务块中可见。

### B. 优先级 (从低到高，高覆盖低)

1. Role `defaults` (最低优先级，易被覆盖)
2. Inventory `group_vars` / `host_vars`
3. Play `vars`
4. Task `vars` / `set_fact`
5. Block `vars`
6. Command line `-e` (最高优先级，强制覆盖)

**在你的场景中**:
- `NODE_TO_ADD` 通常由命令行 `-e` 传入，因此具有最高优先级，确保能覆盖 Inventory 中可能存在的同名变量。
- `ansible_facts` 是由系统自动收集的，优先级很高，但不能被 `-e` 直接覆盖（除非使用 `set_fact` 修改）。

---

## 3. 变量生命周期 (Lifecycle)

### A. Facts (`ansible_facts`)

- **生成**: 在每个 Play 开始时，通过 `setup` 模块或显式调用 `service_facts`, `package_facts` 等生成。
- **缓存**: 如果开启了 `fact_caching`，Facts 会被缓存到磁盘/Redis，下次执行时直接读取，无需重新连接主机收集。
- **失效**:
  - 默认情况下，Facts 在整个 Play 执行期间有效。
  - 如果调用了 `meta: clear_facts`，或者新的 Play 开始且未禁用 fact 收集，Facts 可能会刷新。
  - 在你的文件中，`service_facts` 执行后，`ansible_facts['services']` 立即更新，供后续 `when` 判断使用。

### B. 普通变量

- **持久性**: 在单个 Play 的执行过程中，变量一旦定义（通过 Inventory, vars, set_fact），就一直存在。
- **跨 Play**: 如果多个 Play 在同一个 playbook 文件中，默认情况下变量**不**自动共享，除非使用了 `hostvars` 或者将变量注册为 Facts (`set_fact` 默认行为类似事实，可在后续 Play 中访问)。



## 作用域
* 定义在 inventory 文件、`group_vars/` 或 `host_vars/` 目录中的变量是全局可用的，任何 Play 只要 targeting 对应的主机，都可以访问这些变量。
* 在一个play中定义的变量，一般不跨play可见。