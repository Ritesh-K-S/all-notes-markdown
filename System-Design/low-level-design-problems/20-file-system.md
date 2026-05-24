# 20 - File System (In-Memory)

## 📋 Problem Statement

Design an in-memory file system that supports:
- Creating files and directories
- Navigation (ls, cd, pwd)
- Read/write file content
- Hierarchical directory structure

---

## 📌 Requirements

### Functional Requirements
1. **mkdir** — create directory
2. **touch** — create file
3. **ls** — list directory contents
4. **cd** — change directory
5. **cat** — read file content
6. **write** — write to file
7. **rm** — delete file or directory
8. **find** — search for files by name
9. Support **absolute** and **relative** paths

---

## 💻 Code Implementation (Composite Pattern)

### FileSystem Node (Composite)

```java
public abstract class FileSystemNode {
    protected String name;
    protected Directory parent;
    protected LocalDateTime createdAt;
    protected LocalDateTime modifiedAt;

    public FileSystemNode(String name) {
        this.name = name;
        this.createdAt = LocalDateTime.now();
        this.modifiedAt = LocalDateTime.now();
    }

    public abstract int getSize();
    public abstract String getType();
    public abstract void display(String indent);

    public String getPath() {
        if (parent == null) return "/";
        String parentPath = parent.getPath();
        return parentPath.equals("/") ? "/" + name : parentPath + "/" + name;
    }

    public String getName() { return name; }
    public Directory getParent() { return parent; }
    public void setParent(Directory parent) { this.parent = parent; }
}
```

### File (Leaf)

```java
public class File extends FileSystemNode {
    private StringBuilder content;

    public File(String name) {
        super(name);
        this.content = new StringBuilder();
    }

    public void write(String data) {
        this.content = new StringBuilder(data);
        this.modifiedAt = LocalDateTime.now();
    }

    public void append(String data) {
        this.content.append(data);
        this.modifiedAt = LocalDateTime.now();
    }

    public String read() {
        return content.toString();
    }

    @Override
    public int getSize() { return content.length(); }

    @Override
    public String getType() { return "FILE"; }

    @Override
    public void display(String indent) {
        System.out.println(indent + "📄 " + name + " (" + getSize() + " bytes)");
    }
}
```

### Directory (Composite)

```java
public class Directory extends FileSystemNode {
    private Map<String, FileSystemNode> children;

    public Directory(String name) {
        super(name);
        this.children = new LinkedHashMap<>();
    }

    public void add(FileSystemNode node) {
        if (children.containsKey(node.getName())) {
            throw new FileAlreadyExistsException(node.getName() + " already exists");
        }
        children.put(node.getName(), node);
        node.setParent(this);
    }

    public void remove(String name) {
        if (!children.containsKey(name)) {
            throw new FileNotFoundException(name + " not found");
        }
        children.remove(name);
    }

    public FileSystemNode get(String name) {
        return children.get(name);
    }

    public List<String> list() {
        return new ArrayList<>(children.keySet());
    }

    @Override
    public int getSize() {
        return children.values().stream().mapToInt(FileSystemNode::getSize).sum();
    }

    @Override
    public String getType() { return "DIRECTORY"; }

    @Override
    public void display(String indent) {
        System.out.println(indent + "📁 " + name + "/");
        for (FileSystemNode child : children.values()) {
            child.display(indent + "  ");
        }
    }

    public Map<String, FileSystemNode> getChildren() { return children; }
}
```

### FileSystem Controller

```java
public class FileSystem {
    private Directory root;
    private Directory currentDir;

    public FileSystem() {
        this.root = new Directory("");
        this.currentDir = root;
    }

    // ─── mkdir ──────────────────────────────────
    public void mkdir(String path) {
        String[] parts = parsePath(path);
        Directory dir = navigateToParent(parts);
        String dirName = parts[parts.length - 1];

        Directory newDir = new Directory(dirName);
        dir.add(newDir);
    }

    // ─── touch (create file) ────────────────────
    public void touch(String path) {
        String[] parts = parsePath(path);
        Directory dir = navigateToParent(parts);
        String fileName = parts[parts.length - 1];

        File file = new File(fileName);
        dir.add(file);
    }

    // ─── ls ─────────────────────────────────────
    public List<String> ls(String path) {
        Directory dir = (path == null || path.isEmpty()) ? currentDir : navigateTo(path);
        return dir.list();
    }

    // ─── cd ─────────────────────────────────────
    public void cd(String path) {
        if (path.equals("..")) {
            if (currentDir.getParent() != null) {
                currentDir = currentDir.getParent();
            }
            return;
        }
        if (path.equals("/")) {
            currentDir = root;
            return;
        }
        currentDir = navigateTo(path);
    }

    // ─── pwd ────────────────────────────────────
    public String pwd() {
        return currentDir.getPath();
    }

    // ─── cat (read file) ────────────────────────
    public String cat(String path) {
        FileSystemNode node = resolve(path);
        if (!(node instanceof File)) throw new IllegalArgumentException("Not a file: " + path);
        return ((File) node).read();
    }

    // ─── write to file ──────────────────────────
    public void write(String path, String content) {
        FileSystemNode node = resolve(path);
        if (!(node instanceof File)) throw new IllegalArgumentException("Not a file: " + path);
        ((File) node).write(content);
    }

    // ─── rm ─────────────────────────────────────
    public void rm(String path) {
        String[] parts = parsePath(path);
        Directory parent = navigateToParent(parts);
        parent.remove(parts[parts.length - 1]);
    }

    // ─── find ───────────────────────────────────
    public List<String> find(String name) {
        List<String> results = new ArrayList<>();
        findRecursive(root, name, results);
        return results;
    }

    private void findRecursive(Directory dir, String name, List<String> results) {
        for (FileSystemNode node : dir.getChildren().values()) {
            if (node.getName().contains(name)) {
                results.add(node.getPath());
            }
            if (node instanceof Directory) {
                findRecursive((Directory) node, name, results);
            }
        }
    }

    // ─── tree ───────────────────────────────────
    public void tree() {
        root.display("");
    }

    // ─── Helper methods ─────────────────────────
    private String[] parsePath(String path) {
        return path.split("/");
    }

    private Directory navigateTo(String path) {
        String[] parts = parsePath(path);
        Directory dir = path.startsWith("/") ? root : currentDir;

        for (String part : parts) {
            if (part.isEmpty() || part.equals(".")) continue;
            if (part.equals("..")) {
                dir = dir.getParent() != null ? dir.getParent() : dir;
                continue;
            }
            FileSystemNode node = dir.get(part);
            if (!(node instanceof Directory)) {
                throw new IllegalArgumentException("Not a directory: " + part);
            }
            dir = (Directory) node;
        }
        return dir;
    }

    private Directory navigateToParent(String[] parts) {
        Directory dir = currentDir;
        for (int i = 0; i < parts.length - 1; i++) {
            if (parts[i].isEmpty()) continue;
            FileSystemNode node = dir.get(parts[i]);
            if (!(node instanceof Directory)) {
                throw new IllegalArgumentException("Not a directory: " + parts[i]);
            }
            dir = (Directory) node;
        }
        return dir;
    }

    private FileSystemNode resolve(String path) {
        String[] parts = parsePath(path);
        Directory dir = navigateToParent(parts);
        FileSystemNode node = dir.get(parts[parts.length - 1]);
        if (node == null) throw new FileNotFoundException("Not found: " + path);
        return node;
    }
}
```

---

## 🧪 Usage Example

```java
FileSystem fs = new FileSystem();

fs.mkdir("home");
fs.mkdir("home/user");
fs.cd("home/user");
fs.mkdir("documents");
fs.touch("documents/readme.txt");
fs.write("documents/readme.txt", "Hello, World!");

System.out.println(fs.pwd());                    // /home/user
System.out.println(fs.ls("documents"));          // [readme.txt]
System.out.println(fs.cat("documents/readme.txt")); // Hello, World!

fs.cd("/");
fs.tree();
// 📁 /
//   📁 home/
//     📁 user/
//       📁 documents/
//         📄 readme.txt (13 bytes)

List<String> found = fs.find("readme");
// [/home/user/documents/readme.txt]
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Composite** | Directory contains FileSystemNodes (files or directories) |
| **Iterator** | Traversal for ls, find, tree |

---

## 🔑 Key Takeaways

1. **Composite Pattern** is the core — File (leaf) and Directory (composite) share interface
2. **Path resolution** handles absolute (`/`) and relative paths
3. **Recursive operations** (find, getSize, tree) naturally follow composite structure
4. **LinkedHashMap** preserves insertion order for `ls`
5. Parent pointer enables `cd ..` and `getPath()`
