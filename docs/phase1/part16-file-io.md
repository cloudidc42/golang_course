# Part 16: File I/O ใน Go

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `os` package สำหรับการอ่านและเขียนไฟล์
- ใช้ `bufio` package สำหรับการ buffer I/O
- จัดการ path ด้วย `filepath` package
- เดิน directory tree
- จัดการ file permissions
- สร้างและใช้งาน temporary files
- อ่านและเขียนไฟล์ CSV

---

## 16.1 os Package พื้นฐาน

`os` package เป็น package หลักสำหรับการทำงานกับ operating system รวมถึงการจัดการไฟล์

### 16.1.1 ReadFile และ WriteFile

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // เขียนไฟล์ด้วย os.WriteFile
    content := []byte("สวัสดี Go!\nนี่คือบรรทัดที่สอง\nนี่คือบรรทัดที่สาม\n")
    
    err := os.WriteFile("hello.txt", content, 0644)
    if err != nil {
        fmt.Printf("เขียนไฟล์ไม่สำเร็จ: %v\n", err)
        return
    }
    fmt.Println("เขียนไฟล์สำเร็จ!")
    
    // อ่านไฟล์ด้วย os.ReadFile
    data, err := os.ReadFile("hello.txt")
    if err != nil {
        fmt.Printf("อ่านไฟล์ไม่สำเร็จ: %v\n", err)
        return
    }
    
    fmt.Printf("เนื้อหาในไฟล์:\n%s", data)
    
    // ลบไฟล์หลังใช้งาน
    os.Remove("hello.txt")
}
```

### 16.1.2 OpenFile - เปิดไฟล์แบบ Advanced

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // สร้างไฟล์ใหม่ (ถ้ามีอยู่แล้วจะ truncate)
    f, err := os.OpenFile("data.txt", os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0644)
    if err != nil {
        fmt.Printf("เปิดไฟล์ไม่สำเร็จ: %v\n", err)
        return
    }
    
    // เขียนข้อมูล
    f.WriteString("บรรทัดที่ 1\n")
    f.WriteString("บรรทัดที่ 2\n")
    f.Close()
    
    // เปิดไฟล์แบบ append
    f2, err := os.OpenFile("data.txt", os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        fmt.Printf("เปิดไฟล์ไม่สำเร็จ: %v\n", err)
        return
    }
    defer f2.Close()
    
    f2.WriteString("บรรทัดที่ 3 (เพิ่มภายหลัง)\n")
    
    // อ่านไฟล์
    content, _ := os.ReadFile("data.txt")
    fmt.Println("เนื้อหาทั้งหมด:")
    fmt.Print(string(content))
    
    os.Remove("data.txt")
}
```

### 16.1.3 File Flags ที่ใช้บ่อย

```go
package main

import (
    "fmt"
    "os"
)

func demonstrateFlags() {
    // O_RDONLY - อ่านอย่างเดียว
    // O_WRONLY - เขียนอย่างเดียว
    // O_RDWR   - อ่านและเขียน
    // O_APPEND - เพิ่มต่อท้าย
    // O_CREATE - สร้างถ้าไม่มี
    // O_EXCL   - error ถ้าไฟล์มีอยู่แล้ว (ใช้กับ O_CREATE)
    // O_SYNC   - เปิด synchronous I/O
    // O_TRUNC  - ตัดไฟล์ให้เป็น 0 เมื่อเปิด
    
    // ตัวอย่าง: สร้างไฟล์ใหม่เท่านั้น ถ้ามีอยู่แล้วให้ error
    f, err := os.OpenFile("unique.txt", os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)
    if err != nil {
        fmt.Printf("ไม่สามารถสร้างไฟล์: %v\n", err)
    } else {
        f.WriteString("ไฟล์ใหม่!\n")
        f.Close()
        fmt.Println("สร้างไฟล์ใหม่สำเร็จ")
        os.Remove("unique.txt")
    }
    
    // ตัวอย่าง: เปิดไฟล์อ่าน/เขียน
    os.WriteFile("rw.txt", []byte("initial content"), 0644)
    f2, err := os.OpenFile("rw.txt", os.O_RDWR, 0644)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    defer f2.Close()
    defer os.Remove("rw.txt")
    
    // เขียนทับตั้งแต่ต้น
    f2.WriteString("NEW ")
    
    // กลับไปต้นไฟล์
    f2.Seek(0, 0)
    
    buf := make([]byte, 100)
    n, _ := f2.Read(buf)
    fmt.Printf("อ่านได้: %s\n", buf[:n])
}

func main() {
    demonstrateFlags()
}
```

### 16.1.4 File Info และ Stat

```go
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    // สร้างไฟล์ทดสอบ
    os.WriteFile("test.txt", []byte("Hello, World!"), 0644)
    defer os.Remove("test.txt")
    
    // ดูข้อมูลไฟล์
    info, err := os.Stat("test.txt")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    fmt.Printf("ชื่อไฟล์: %s\n", info.Name())
    fmt.Printf("ขนาด: %d bytes\n", info.Size())
    fmt.Printf("เป็น directory: %v\n", info.IsDir())
    fmt.Printf("Mode: %v\n", info.Mode())
    fmt.Printf("แก้ไขล่าสุด: %v\n", info.ModTime().Format(time.RFC3339))
    
    // ตรวจสอบว่าไฟล์มีอยู่หรือไม่
    checkFile := func(path string) {
        _, err := os.Stat(path)
        if os.IsNotExist(err) {
            fmt.Printf("ไฟล์ %s ไม่มีอยู่\n", path)
        } else if err != nil {
            fmt.Printf("error: %v\n", err)
        } else {
            fmt.Printf("ไฟล์ %s มีอยู่\n", path)
        }
    }
    
    checkFile("test.txt")
    checkFile("notexist.txt")
}
```

---

## 16.2 bufio Package

`bufio` package ให้การ buffer I/O ที่มีประสิทธิภาพสูงกว่าการอ่านเขียนโดยตรง

### 16.2.1 bufio.Scanner - อ่านทีละบรรทัด

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    // สร้างไฟล์ทดสอบ
    content := `บรรทัดที่ 1: สวัสดีชาวโลก
บรรทัดที่ 2: Go is awesome
บรรทัดที่ 3: เรียน Go กัน
บรรทัดที่ 4: มีความสุขกับการเขียนโค้ด
`
    os.WriteFile("lines.txt", []byte(content), 0644)
    defer os.Remove("lines.txt")
    
    // เปิดไฟล์
    f, err := os.Open("lines.txt")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    defer f.Close()
    
    // สร้าง Scanner
    scanner := bufio.NewScanner(f)
    
    lineNum := 1
    for scanner.Scan() {
        line := scanner.Text()
        fmt.Printf("%d: %s\n", lineNum, line)
        lineNum++
    }
    
    if err := scanner.Err(); err != nil {
        fmt.Printf("error scanning: %v\n", err)
    }
    
    // อ่านจาก string
    fmt.Println("\n--- อ่านจาก string ---")
    r := strings.NewReader("word1 word2 word3\nword4 word5\n")
    scanner2 := bufio.NewScanner(r)
    scanner2.Split(bufio.ScanWords) // แยกด้วย word แทน line
    
    for scanner2.Scan() {
        fmt.Printf("word: %s\n", scanner2.Text())
    }
}
```

### 16.2.2 bufio.Reader

```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    // สร้างไฟล์ทดสอบ
    content := "Hello, World!\nSecond line\nThird line\n"
    os.WriteFile("read_test.txt", []byte(content), 0644)
    defer os.Remove("read_test.txt")
    
    f, _ := os.Open("read_test.txt")
    defer f.Close()
    
    reader := bufio.NewReader(f)
    
    // อ่านทีละบรรทัด (รวม delimiter)
    for {
        line, err := reader.ReadString('\n')
        if len(line) > 0 {
            fmt.Printf("อ่านได้: %q\n", line)
        }
        if err != nil {
            break
        }
    }
    
    // ReadByte
    f2, _ := os.Open("read_test.txt")
    defer f2.Close()
    
    reader2 := bufio.NewReader(f2)
    fmt.Println("\nอ่านทีละ byte:")
    for i := 0; i < 5; i++ {
        b, err := reader2.ReadByte()
        if err != nil {
            break
        }
        fmt.Printf("byte: %c (%d)\n", b, b)
    }
    
    // Peek - ดูข้อมูลโดยไม่ consume
    reader2.Seek(0, 0) // reset
    f3, _ := os.Open("read_test.txt")
    defer f3.Close()
    reader3 := bufio.NewReader(f3)
    
    peeked, _ := reader3.Peek(5)
    fmt.Printf("\nPeek 5 bytes: %q\n", peeked)
    
    // อ่านจริงๆ
    buf := make([]byte, 5)
    n, _ := reader3.Read(buf)
    fmt.Printf("Read 5 bytes: %q\n", buf[:n])
}
```

### 16.2.3 bufio.Writer

```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    f, err := os.Create("output.txt")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    defer f.Close()
    defer os.Remove("output.txt")
    
    // สร้าง buffered writer
    writer := bufio.NewWriter(f)
    
    // เขียนข้อมูล (ถูก buffer ไว้ก่อน)
    writer.WriteString("บรรทัดที่ 1\n")
    writer.WriteString("บรรทัดที่ 2\n")
    
    // เขียน byte
    writer.WriteByte('\n')
    
    // เขียน rune
    writer.WriteRune('🚀')
    writer.WriteRune('\n')
    
    // fmt.Fprintf กับ bufio.Writer
    fmt.Fprintf(writer, "ตัวเลข: %d\n", 42)
    fmt.Fprintf(writer, "Float: %.2f\n", 3.14)
    
    // ต้อง Flush เพื่อให้ข้อมูลที่ buffer ถูกเขียนจริงๆ
    err = writer.Flush()
    if err != nil {
        fmt.Printf("flush error: %v\n", err)
        return
    }
    
    // อ่านกลับมาตรวจสอบ
    content, _ := os.ReadFile("output.txt")
    fmt.Println("เนื้อหาในไฟล์:")
    fmt.Print(string(content))
    
    fmt.Printf("\nขนาด buffer: %d\n", writer.Size())
    fmt.Printf("ข้อมูลที่ยังไม่ flush: %d bytes\n", writer.Buffered())
}
```

### 16.2.4 Buffered I/O Performance

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "time"
)

func writeUnbuffered(filename string, n int) time.Duration {
    f, _ := os.Create(filename)
    defer f.Close()
    
    start := time.Now()
    for i := 0; i < n; i++ {
        fmt.Fprintf(f, "line %d\n", i)
    }
    return time.Since(start)
}

func writeBuffered(filename string, n int) time.Duration {
    f, _ := os.Create(filename)
    defer f.Close()
    
    w := bufio.NewWriter(f)
    
    start := time.Now()
    for i := 0; i < n; i++ {
        fmt.Fprintf(w, "line %d\n", i)
    }
    w.Flush()
    return time.Since(start)
}

func main() {
    n := 10000
    
    t1 := writeUnbuffered("unbuffered.txt", n)
    t2 := writeBuffered("buffered.txt", n)
    
    fmt.Printf("Unbuffered: %v\n", t1)
    fmt.Printf("Buffered: %v\n", t2)
    fmt.Printf("Buffered เร็วกว่า: %.2fx\n", float64(t1)/float64(t2))
    
    os.Remove("unbuffered.txt")
    os.Remove("buffered.txt")
}
```

---

## 16.3 filepath Package

`filepath` package จัดการ file path ข้ามระบบปฏิบัติการ

### 16.3.1 Path Operations

```go
package main

import (
    "fmt"
    "path/filepath"
)

func main() {
    // Join - เชื่อม path components
    p := filepath.Join("home", "user", "documents", "file.txt")
    fmt.Println("Join:", p)
    
    // Dir - ดึง directory
    dir := filepath.Dir("/home/user/documents/file.txt")
    fmt.Println("Dir:", dir)
    
    // Base - ดึง filename
    base := filepath.Base("/home/user/documents/file.txt")
    fmt.Println("Base:", base)
    
    // Ext - ดึง extension
    ext := filepath.Ext("/home/user/documents/file.txt")
    fmt.Println("Ext:", ext)
    
    // Split - แยก dir และ file
    d, f := filepath.Split("/home/user/documents/file.txt")
    fmt.Printf("Split: dir=%q, file=%q\n", d, f)
    
    // Abs - path สมบูรณ์
    abs, err := filepath.Abs("./relative/path")
    if err == nil {
        fmt.Println("Abs:", abs)
    }
    
    // Rel - relative path
    base2, _ := filepath.Abs("/home/user")
    target, _ := filepath.Abs("/home/user/documents/file.txt")
    rel, err := filepath.Rel(base2, target)
    if err == nil {
        fmt.Println("Rel:", rel)
    }
    
    // Match - pattern matching
    matched, _ := filepath.Match("*.txt", "file.txt")
    fmt.Println("Match *.txt:", matched)
    
    matched2, _ := filepath.Match("*.go", "file.txt")
    fmt.Println("Match *.go:", matched2)
    
    // Clean - normalize path
    dirty := filepath.Join("/home", "user", "..", "user", ".", "documents")
    clean := filepath.Clean(dirty)
    fmt.Println("Clean:", clean)
}
```

### 16.3.2 Walking Directories

```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
)

func main() {
    // สร้าง directory structure ทดสอบ
    os.MkdirAll("testdir/subdir1/deep", 0755)
    os.MkdirAll("testdir/subdir2", 0755)
    os.WriteFile("testdir/file1.txt", []byte("f1"), 0644)
    os.WriteFile("testdir/file2.go", []byte("f2"), 0644)
    os.WriteFile("testdir/subdir1/file3.txt", []byte("f3"), 0644)
    os.WriteFile("testdir/subdir1/deep/file4.md", []byte("f4"), 0644)
    os.WriteFile("testdir/subdir2/file5.txt", []byte("f5"), 0644)
    defer os.RemoveAll("testdir")
    
    fmt.Println("=== เดินดู directory ทั้งหมด ===")
    err := filepath.Walk("testdir", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        indent := ""
        rel, _ := filepath.Rel("testdir", path)
        for _, c := range rel {
            if c == filepath.Separator {
                indent += "  "
            }
        }
        
        if info.IsDir() {
            fmt.Printf("%s📁 %s/\n", indent, info.Name())
        } else {
            fmt.Printf("%s📄 %s (%d bytes)\n", indent, info.Name(), info.Size())
        }
        return nil
    })
    
    if err != nil {
        fmt.Printf("Walk error: %v\n", err)
    }
    
    // หาเฉพาะไฟล์ .txt
    fmt.Println("\n=== ไฟล์ .txt ทั้งหมด ===")
    filepath.Walk("testdir", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if !info.IsDir() && filepath.Ext(path) == ".txt" {
            fmt.Println(path)
        }
        return nil
    })
    
    // ข้าม directory บางตัว
    fmt.Println("\n=== ข้าม subdir2 ===")
    filepath.Walk("testdir", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if info.IsDir() && info.Name() == "subdir2" {
            return filepath.SkipDir // ข้ามทั้ง directory นี้
        }
        fmt.Println(path)
        return nil
    })
}
```

### 16.3.3 WalkDir (Go 1.16+)

```go
package main

import (
    "fmt"
    "io/fs"
    "os"
    "path/filepath"
)

func main() {
    // สร้าง directory structure
    os.MkdirAll("project/src", 0755)
    os.MkdirAll("project/tests", 0755)
    os.MkdirAll("project/.git", 0755)
    os.WriteFile("project/main.go", []byte("package main"), 0644)
    os.WriteFile("project/src/util.go", []byte("package src"), 0644)
    os.WriteFile("project/tests/main_test.go", []byte("package main"), 0644)
    defer os.RemoveAll("project")
    
    // WalkDir มีประสิทธิภาพดีกว่า Walk
    // เพราะไม่ต้อง Stat แต่ละ entry
    fmt.Println("=== WalkDir ===")
    filepath.WalkDir("project", func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }
        
        // ข้ามไดเรกทอรีที่ขึ้นต้นด้วย .
        if d.IsDir() && len(d.Name()) > 0 && d.Name()[0] == '.' {
            fmt.Printf("ข้าม: %s\n", path)
            return filepath.SkipDir
        }
        
        info, _ := d.Info()
        fileType := "ไฟล์"
        if d.IsDir() {
            fileType = "โฟลเดอร์"
        }
        
        fmt.Printf("[%s] %s (%v)\n", fileType, path, info.Mode())
        return nil
    })
    
    // ใช้ Glob pattern
    fmt.Println("\n=== Glob ===")
    goFiles, _ := filepath.Glob("project/**/*.go")
    fmt.Println("Go files:", goFiles)
    
    // os.ReadDir - อ่าน directory เดียว
    fmt.Println("\n=== ReadDir ===")
    entries, _ := os.ReadDir("project")
    for _, e := range entries {
        fmt.Printf("  %s (dir: %v)\n", e.Name(), e.IsDir())
    }
}
```

---

## 16.4 File Permissions

### 16.4.1 Unix Permissions

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // สร้างไฟล์ด้วย permissions ต่างๆ
    // Permission bits: rwxrwxrwx
    // 0755 = rwxr-xr-x (owner: rwx, group: r-x, others: r-x)
    // 0644 = rw-r--r-- (owner: rw-, group: r--, others: r--)
    // 0600 = rw------- (owner: rw-, group: ---, others: ---)
    // 0777 = rwxrwxrwx (ทุกคนสามารถ rwx)
    
    // สร้างไฟล์ด้วย permission 0644
    os.WriteFile("public.txt", []byte("public content"), 0644)
    defer os.Remove("public.txt")
    
    // สร้างไฟล์ private
    os.WriteFile("private.txt", []byte("private content"), 0600)
    defer os.Remove("private.txt")
    
    // สร้างไฟล์ executable
    os.WriteFile("script.sh", []byte("#!/bin/bash\necho hello"), 0755)
    defer os.Remove("script.sh")
    
    // ดู permissions
    files := []string{"public.txt", "private.txt", "script.sh"}
    for _, f := range files {
        info, err := os.Stat(f)
        if err != nil {
            continue
        }
        fmt.Printf("%s: %v\n", f, info.Mode())
    }
    
    // เปลี่ยน permissions
    err := os.Chmod("public.txt", 0400) // read-only
    if err != nil {
        fmt.Printf("chmod error: %v\n", err)
    }
    
    info, _ := os.Stat("public.txt")
    fmt.Printf("\nหลัง chmod: public.txt = %v\n", info.Mode())
    
    // กลับเป็น writable ก่อน remove
    os.Chmod("public.txt", 0644)
    
    // สร้าง directory ด้วย permissions
    os.Mkdir("mydir", 0755)
    defer os.Remove("mydir")
    
    dirInfo, _ := os.Stat("mydir")
    fmt.Printf("Directory: %v\n", dirInfo.Mode())
}
```

### 16.4.2 Checking Permissions

```go
package main

import (
    "fmt"
    "os"
)

func canRead(path string) bool {
    f, err := os.Open(path)
    if err != nil {
        return false
    }
    f.Close()
    return true
}

func canWrite(path string) bool {
    // ลองเปิดด้วย write mode
    f, err := os.OpenFile(path, os.O_WRONLY, 0)
    if err != nil {
        return false
    }
    f.Close()
    return true
}

func main() {
    // สร้างไฟล์ต่างๆ
    os.WriteFile("readable.txt", []byte("hello"), 0444) // read-only
    defer os.Remove("readable.txt")
    
    os.WriteFile("writable.txt", []byte("hello"), 0644) // read-write
    defer os.Remove("writable.txt")
    
    files := []string{"readable.txt", "writable.txt", "notexist.txt"}
    
    for _, f := range files {
        read := canRead(f)
        write := canWrite(f)
        fmt.Printf("%s - อ่านได้: %v, เขียนได้: %v\n", f, read, write)
    }
    
    // ตรวจสอบด้วย os.Access (ใช้ syscall)
    // หรือใช้ os.Stat + Mode bits
    checkMode := func(path string) {
        info, err := os.Stat(path)
        if err != nil {
            fmt.Printf("%s: ไม่มีไฟล์\n", path)
            return
        }
        
        mode := info.Mode()
        fmt.Printf("%s:\n", path)
        fmt.Printf("  IsDir: %v\n", mode.IsDir())
        fmt.Printf("  IsRegular: %v\n", mode.IsRegular())
        fmt.Printf("  Perm: %v\n", mode.Perm())
        fmt.Printf("  Owner read: %v\n", mode&0400 != 0)
        fmt.Printf("  Owner write: %v\n", mode&0200 != 0)
        fmt.Printf("  Owner exec: %v\n", mode&0100 != 0)
    }
    
    fmt.Println()
    checkMode("readable.txt")
    checkMode("writable.txt")
    
    // restore permissions
    os.Chmod("readable.txt", 0644)
}
```

---

## 16.5 Temporary Files

### 16.5.1 os.CreateTemp และ os.MkdirTemp

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // สร้าง temporary file
    // pattern: "prefix-*.suffix"
    tmpFile, err := os.CreateTemp("", "myapp-*.txt")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    defer os.Remove(tmpFile.Name()) // ลบเมื่อเสร็จ
    
    fmt.Printf("Temp file: %s\n", tmpFile.Name())
    
    // เขียนข้อมูลลง temp file
    tmpFile.WriteString("ข้อมูลชั่วคราว\n")
    tmpFile.WriteString("บรรทัดที่ 2\n")
    tmpFile.Close()
    
    // อ่านกลับมา
    content, _ := os.ReadFile(tmpFile.Name())
    fmt.Printf("เนื้อหา: %s", content)
    
    // สร้าง temp file ใน directory เฉพาะ
    tmpFile2, err := os.CreateTemp("/tmp", "cache-*.dat")
    if err != nil {
        fmt.Printf("error: %v\n", err)
    } else {
        defer os.Remove(tmpFile2.Name())
        fmt.Printf("Temp file 2: %s\n", tmpFile2.Name())
        tmpFile2.Close()
    }
    
    // สร้าง temporary directory
    tmpDir, err := os.MkdirTemp("", "myapp-*")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    defer os.RemoveAll(tmpDir) // ลบทั้ง directory
    
    fmt.Printf("Temp dir: %s\n", tmpDir)
    
    // สร้างไฟล์ใน temp dir
    os.WriteFile(tmpDir+"/data.txt", []byte("temp data"), 0644)
    os.WriteFile(tmpDir+"/config.json", []byte("{}"), 0644)
    
    entries, _ := os.ReadDir(tmpDir)
    fmt.Println("ไฟล์ใน temp dir:")
    for _, e := range entries {
        fmt.Printf("  - %s\n", e.Name())
    }
}
```

---

## 16.6 CSV Reading/Writing

### 16.6.1 อ่าน CSV

```go
package main

import (
    "encoding/csv"
    "fmt"
    "os"
    "strconv"
)

type Student struct {
    Name  string
    Age   int
    Score float64
    Grade string
}

func main() {
    // สร้างไฟล์ CSV ทดสอบ
    csvContent := `ชื่อ,อายุ,คะแนน,เกรด
สมชาย ใจดี,20,85.5,A
สมหญิง รักเรียน,21,72.0,B
วิชัย เก่งมาก,19,95.0,A+
มานี มีสุข,22,60.5,C
`
    os.WriteFile("students.csv", []byte(csvContent), 0644)
    defer os.Remove("students.csv")
    
    // เปิดไฟล์
    f, err := os.Open("students.csv")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    defer f.Close()
    
    // สร้าง CSV reader
    reader := csv.NewReader(f)
    reader.Comment = '#' // บรรทัดที่ขึ้นต้นด้วย # คือ comment
    
    // อ่าน header
    header, err := reader.Read()
    if err != nil {
        fmt.Printf("error reading header: %v\n", err)
        return
    }
    fmt.Println("Header:", header)
    
    // อ่าน records ที่เหลือ
    var students []Student
    
    records, err := reader.ReadAll()
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    
    for _, record := range records {
        age, _ := strconv.Atoi(record[1])
        score, _ := strconv.ParseFloat(record[2], 64)
        
        students = append(students, Student{
            Name:  record[0],
            Age:   age,
            Score: score,
            Grade: record[3],
        })
    }
    
    fmt.Println("\nรายชื่อนักเรียน:")
    for _, s := range students {
        fmt.Printf("  %s (อายุ %d) - คะแนน %.1f เกรด %s\n",
            s.Name, s.Age, s.Score, s.Grade)
    }
}
```

### 16.6.2 เขียน CSV

```go
package main

import (
    "encoding/csv"
    "fmt"
    "os"
    "strconv"
)

func main() {
    // ข้อมูลที่จะเขียน
    products := [][]string{
        {"รหัส", "ชื่อสินค้า", "ราคา", "จำนวน"},
        {"P001", "iPhone 15", "35000", "50"},
        {"P002", "Samsung S24", "28000", "75"},
        {"P003", "MacBook Pro", "85000", "20"},
        {"P004", "iPad Air", "22000", "100"},
    }
    
    // สร้างไฟล์
    f, err := os.Create("products.csv")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    defer f.Close()
    defer os.Remove("products.csv")
    
    // สร้าง CSV writer
    writer := csv.NewWriter(f)
    defer writer.Flush() // ต้อง flush เมื่อเสร็จ
    
    // เขียนทีละ record
    for _, record := range products {
        if err := writer.Write(record); err != nil {
            fmt.Printf("error writing: %v\n", err)
            return
        }
    }
    
    // Flush และตรวจสอบ error
    writer.Flush()
    if err := writer.Error(); err != nil {
        fmt.Printf("error: %v\n", err)
    }
    
    // อ่านกลับมาตรวจสอบ
    content, _ := os.ReadFile("products.csv")
    fmt.Println("CSV ที่สร้าง:")
    fmt.Print(string(content))
    
    // เขียนด้วย custom settings
    f2, _ := os.Create("products_tab.csv")
    defer f2.Close()
    defer os.Remove("products_tab.csv")
    
    writer2 := csv.NewWriter(f2)
    writer2.Comma = '\t' // ใช้ tab แทน comma
    
    for _, p := range products {
        writer2.Write(p)
    }
    writer2.Flush()
    
    fmt.Println("\nCSV แบบ Tab-separated:")
    content2, _ := os.ReadFile("products_tab.csv")
    fmt.Print(string(content2))
    
    // ตัวอย่าง: คำนวณและเขียน summary
    f3, _ := os.Create("summary.csv")
    defer f3.Close()
    defer os.Remove("summary.csv")
    
    writer3 := csv.NewWriter(f3)
    writer3.Write([]string{"ชื่อสินค้า", "ราคา", "จำนวน", "มูลค่ารวม"})
    
    for i, p := range products[1:] { // ข้าม header
        price, _ := strconv.ParseFloat(p[2], 64)
        qty, _ := strconv.Atoi(p[3])
        total := price * float64(qty)
        
        writer3.Write([]string{
            p[1],
            p[2],
            p[3],
            fmt.Sprintf("%.2f", total),
        })
        
        _ = i // suppress unused variable
    }
    writer3.Flush()
    
    content3, _ := os.ReadFile("summary.csv")
    fmt.Println("\nSummary CSV:")
    fmt.Print(string(content3))
}
```

### 16.6.3 อ่าน CSV ด้วย Struct

```go
package main

import (
    "encoding/csv"
    "fmt"
    "os"
    "strconv"
    "strings"
)

type Employee struct {
    ID         int
    Name       string
    Department string
    Salary     float64
    Active     bool
}

func parseEmployee(record []string) (Employee, error) {
    if len(record) < 5 {
        return Employee{}, fmt.Errorf("invalid record length: %d", len(record))
    }
    
    id, err := strconv.Atoi(strings.TrimSpace(record[0]))
    if err != nil {
        return Employee{}, fmt.Errorf("invalid id: %v", err)
    }
    
    salary, err := strconv.ParseFloat(strings.TrimSpace(record[3]), 64)
    if err != nil {
        return Employee{}, fmt.Errorf("invalid salary: %v", err)
    }
    
    active := strings.TrimSpace(strings.ToLower(record[4])) == "true"
    
    return Employee{
        ID:         id,
        Name:       strings.TrimSpace(record[1]),
        Department: strings.TrimSpace(record[2]),
        Salary:     salary,
        Active:     active,
    }, nil
}

func main() {
    csvData := `id,name,department,salary,active
1,สมชาย ใจดี,Engineering,75000,true
2,สมหญิง รักเรียน,Marketing,65000,true
3,วิชัย เก่งมาก,Engineering,85000,false
4,มานี มีสุข,HR,55000,true
5,ประดับ ดีมาก,Finance,70000,true
`
    
    reader := csv.NewReader(strings.NewReader(csvData))
    
    // อ่าน header
    headers, _ := reader.Read()
    fmt.Println("Headers:", headers)
    
    var employees []Employee
    var errors []error
    
    for {
        record, err := reader.Read()
        if err != nil {
            break
        }
        
        emp, err := parseEmployee(record)
        if err != nil {
            errors = append(errors, err)
            continue
        }
        employees = append(employees, emp)
    }
    
    // แสดงผล
    fmt.Printf("\nพนักงานทั้งหมด: %d คน\n", len(employees))
    for _, e := range employees {
        status := "✓"
        if !e.Active {
            status = "✗"
        }
        fmt.Printf("[%s] ID:%d %s (%s) - ฿%.0f\n",
            status, e.ID, e.Name, e.Department, e.Salary)
    }
    
    // สถิติ
    var total float64
    active := 0
    for _, e := range employees {
        total += e.Salary
        if e.Active {
            active++
        }
    }
    fmt.Printf("\nเงินเดือนรวม: ฿%.0f\n", total)
    fmt.Printf("พนักงาน Active: %d คน\n", active)
    
    // เขียน filtered CSV
    _ = os.Stdout // just to show we could write
    
    if len(errors) > 0 {
        fmt.Printf("\nErrors: %v\n", errors)
    }
}
```

---

## 16.7 Advanced File Operations

### 16.7.1 Copy Files

```go
package main

import (
    "fmt"
    "io"
    "os"
)

func copyFile(src, dst string) (int64, error) {
    // เปิดไฟล์ต้นทาง
    srcFile, err := os.Open(src)
    if err != nil {
        return 0, fmt.Errorf("เปิดไฟล์ต้นทางไม่ได้: %w", err)
    }
    defer srcFile.Close()
    
    // สร้างไฟล์ปลายทาง
    dstFile, err := os.Create(dst)
    if err != nil {
        return 0, fmt.Errorf("สร้างไฟล์ปลายทางไม่ได้: %w", err)
    }
    defer dstFile.Close()
    
    // copy
    bytes, err := io.Copy(dstFile, srcFile)
    if err != nil {
        return 0, fmt.Errorf("copy ไม่สำเร็จ: %w", err)
    }
    
    // copy permissions
    srcInfo, err := os.Stat(src)
    if err == nil {
        os.Chmod(dst, srcInfo.Mode())
    }
    
    return bytes, nil
}

func copyWithProgress(src, dst string) error {
    srcFile, err := os.Open(src)
    if err != nil {
        return err
    }
    defer srcFile.Close()
    
    srcInfo, _ := os.Stat(src)
    total := srcInfo.Size()
    
    dstFile, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer dstFile.Close()
    
    buf := make([]byte, 32*1024) // 32KB buffer
    var copied int64
    
    for {
        n, err := srcFile.Read(buf)
        if n > 0 {
            written, werr := dstFile.Write(buf[:n])
            copied += int64(written)
            
            // แสดง progress
            progress := float64(copied) / float64(total) * 100
            fmt.Printf("\rCopying... %.1f%% (%d/%d bytes)", progress, copied, total)
            
            if werr != nil {
                return werr
            }
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
    }
    fmt.Println()
    return nil
}

func main() {
    // สร้างไฟล์ต้นทาง
    os.WriteFile("source.txt", []byte("Hello, this is source file content!\nLine 2\nLine 3"), 0644)
    defer os.Remove("source.txt")
    defer os.Remove("destination.txt")
    defer os.Remove("destination2.txt")
    
    // copy ธรรมดา
    n, err := copyFile("source.txt", "destination.txt")
    if err != nil {
        fmt.Printf("error: %v\n", err)
        return
    }
    fmt.Printf("Copy สำเร็จ: %d bytes\n", n)
    
    // ตรวจสอบ
    srcContent, _ := os.ReadFile("source.txt")
    dstContent, _ := os.ReadFile("destination.txt")
    fmt.Printf("Source: %q\n", srcContent)
    fmt.Printf("Dest:   %q\n", dstContent)
    
    // copy with progress
    // สร้างไฟล์ใหญ่ขึ้น
    bigContent := make([]byte, 100*1024) // 100KB
    for i := range bigContent {
        bigContent[i] = byte(i % 256)
    }
    os.WriteFile("big_source.bin", bigContent, 0644)
    defer os.Remove("big_source.bin")
    defer os.Remove("big_dest.bin")
    
    err = copyWithProgress("big_source.bin", "big_dest.bin")
    if err != nil {
        fmt.Printf("error: %v\n", err)
    } else {
        fmt.Println("Copy with progress สำเร็จ!")
    }
}
```

### 16.7.2 Reading Large Files Efficiently

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func processLargeFile(filename string, processor func(string) string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    
    // Output file
    outFile, err := os.Create(filename + ".out")
    if err != nil {
        return err
    }
    defer outFile.Close()
    
    scanner := bufio.NewScanner(f)
    writer := bufio.NewWriter(outFile)
    defer writer.Flush()
    
    lineNum := 0
    for scanner.Scan() {
        line := scanner.Text()
        processed := processor(line)
        fmt.Fprintln(writer, processed)
        lineNum++
    }
    
    fmt.Printf("ประมวลผล %d บรรทัด\n", lineNum)
    return scanner.Err()
}

func countLines(filename string) (int, error) {
    f, err := os.Open(filename)
    if err != nil {
        return 0, err
    }
    defer f.Close()
    
    scanner := bufio.NewScanner(f)
    count := 0
    for scanner.Scan() {
        count++
    }
    return count, scanner.Err()
}

func searchInFile(filename, pattern string) ([]string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    
    var results []string
    scanner := bufio.NewScanner(f)
    lineNum := 0
    
    for scanner.Scan() {
        lineNum++
        line := scanner.Text()
        if strings.Contains(line, pattern) {
            results = append(results, fmt.Sprintf("%d: %s", lineNum, line))
        }
    }
    
    return results, scanner.Err()
}

func main() {
    // สร้างไฟล์ทดสอบขนาดใหญ่
    f, _ := os.Create("large.txt")
    w := bufio.NewWriter(f)
    
    for i := 1; i <= 1000; i++ {
        if i%100 == 0 {
            fmt.Fprintf(w, "บรรทัดที่ %d - นี่คือบรรทัดพิเศษ MARKER\n", i)
        } else {
            fmt.Fprintf(w, "บรรทัดที่ %d - เนื้อหาทั่วไป data=%d\n", i, i*i)
        }
    }
    w.Flush()
    f.Close()
    
    defer os.Remove("large.txt")
    defer os.Remove("large.txt.out")
    
    // นับบรรทัด
    count, _ := countLines("large.txt")
    fmt.Printf("จำนวนบรรทัด: %d\n", count)
    
    // ค้นหา
    results, _ := searchInFile("large.txt", "MARKER")
    fmt.Printf("\nพบ MARKER %d บรรทัด:\n", len(results))
    for _, r := range results[:3] { // แสดง 3 อันแรก
        fmt.Println(" ", r)
    }
    
    // ประมวลผล
    processLargeFile("large.txt", func(line string) string {
        return strings.ToUpper(line)
    })
    
    // ตรวจสอบ output
    outLines, _ := countLines("large.txt.out")
    fmt.Printf("\nOutput: %d บรรทัด\n", outLines)
}
```

---

## 16.8 Workshop: File Manager

```go
package main

import (
    "bufio"
    "encoding/csv"
    "fmt"
    "os"
    "path/filepath"
    "sort"
    "strconv"
    "strings"
    "time"
)

type FileInfo struct {
    Name    string
    Path    string
    Size    int64
    ModTime time.Time
    IsDir   bool
}

type FileManager struct {
    rootDir string
}

func NewFileManager(root string) *FileManager {
    return &FileManager{rootDir: root}
}

func (fm *FileManager) List(dir string) ([]FileInfo, error) {
    entries, err := os.ReadDir(filepath.Join(fm.rootDir, dir))
    if err != nil {
        return nil, err
    }
    
    var files []FileInfo
    for _, e := range entries {
        info, err := e.Info()
        if err != nil {
            continue
        }
        
        files = append(files, FileInfo{
            Name:    e.Name(),
            Path:    filepath.Join(dir, e.Name()),
            Size:    info.Size(),
            ModTime: info.ModTime(),
            IsDir:   e.IsDir(),
        })
    }
    
    // Sort: directories first, then by name
    sort.Slice(files, func(i, j int) bool {
        if files[i].IsDir != files[j].IsDir {
            return files[i].IsDir
        }
        return files[i].Name < files[j].Name
    })
    
    return files, nil
}

func (fm *FileManager) Search(pattern string) ([]FileInfo, error) {
    var results []FileInfo
    
    err := filepath.Walk(fm.rootDir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return nil // skip errors
        }
        
        matched, _ := filepath.Match(pattern, info.Name())
        if matched {
            rel, _ := filepath.Rel(fm.rootDir, path)
            results = append(results, FileInfo{
                Name:    info.Name(),
                Path:    rel,
                Size:    info.Size(),
                ModTime: info.ModTime(),
                IsDir:   info.IsDir(),
            })
        }
        return nil
    })
    
    return results, err
}

func (fm *FileManager) Read(path string) (string, error) {
    fullPath := filepath.Join(fm.rootDir, path)
    content, err := os.ReadFile(fullPath)
    if err != nil {
        return "", err
    }
    return string(content), nil
}

func (fm *FileManager) Write(path, content string) error {
    fullPath := filepath.Join(fm.rootDir, path)
    
    // สร้าง directory ถ้าไม่มี
    dir := filepath.Dir(fullPath)
    if err := os.MkdirAll(dir, 0755); err != nil {
        return err
    }
    
    return os.WriteFile(fullPath, []byte(content), 0644)
}

func (fm *FileManager) Delete(path string) error {
    fullPath := filepath.Join(fm.rootDir, path)
    info, err := os.Stat(fullPath)
    if err != nil {
        return err
    }
    
    if info.IsDir() {
        return os.RemoveAll(fullPath)
    }
    return os.Remove(fullPath)
}

func (fm *FileManager) Stats() map[string]interface{} {
    stats := map[string]interface{}{
        "total_files": 0,
        "total_dirs":  0,
        "total_size":  int64(0),
        "extensions":  map[string]int{},
    }
    
    filepath.Walk(fm.rootDir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return nil
        }
        if info.IsDir() {
            stats["total_dirs"] = stats["total_dirs"].(int) + 1
        } else {
            stats["total_files"] = stats["total_files"].(int) + 1
            stats["total_size"] = stats["total_size"].(int64) + info.Size()
            
            ext := strings.ToLower(filepath.Ext(info.Name()))
            if ext != "" {
                extMap := stats["extensions"].(map[string]int)
                extMap[ext]++
            }
        }
        return nil
    })
    
    return stats
}

func (fm *FileManager) ExportToCSV(outputPath string) error {
    f, err := os.Create(outputPath)
    if err != nil {
        return err
    }
    defer f.Close()
    
    writer := csv.NewWriter(f)
    defer writer.Flush()
    
    // Header
    writer.Write([]string{"ชื่อ", "Path", "ขนาด (bytes)", "วันแก้ไขล่าสุด", "ประเภท"})
    
    return filepath.Walk(fm.rootDir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return nil
        }
        
        rel, _ := filepath.Rel(fm.rootDir, path)
        fileType := "file"
        if info.IsDir() {
            fileType = "directory"
        }
        
        return writer.Write([]string{
            info.Name(),
            rel,
            strconv.FormatInt(info.Size(), 10),
            info.ModTime().Format("2006-01-02 15:04:05"),
            fileType,
        })
    })
}

func main() {
    // สร้าง workspace
    workspace := "workspace"
    os.MkdirAll(workspace+"/src", 0755)
    os.MkdirAll(workspace+"/docs", 0755)
    os.MkdirAll(workspace+"/tests", 0755)
    
    os.WriteFile(workspace+"/main.go", []byte("package main\n\nfunc main() {}\n"), 0644)
    os.WriteFile(workspace+"/src/util.go", []byte("package src\n"), 0644)
    os.WriteFile(workspace+"/src/helper.go", []byte("package src\n"), 0644)
    os.WriteFile(workspace+"/docs/README.md", []byte("# My Project\n"), 0644)
    os.WriteFile(workspace+"/tests/main_test.go", []byte("package main\n"), 0644)
    
    defer os.RemoveAll(workspace)
    defer os.Remove("workspace.csv")
    
    // สร้าง FileManager
    fm := NewFileManager(workspace)
    
    // List root
    fmt.Println("=== รายการใน root ===")
    files, _ := fm.List(".")
    for _, f := range files {
        icon := "📄"
        if f.IsDir {
            icon = "📁"
        }
        fmt.Printf("%s %s\n", icon, f.Name)
    }
    
    // List subdirectory
    fmt.Println("\n=== รายการใน src ===")
    srcFiles, _ := fm.List("src")
    for _, f := range srcFiles {
        fmt.Printf("  - %s (%d bytes)\n", f.Name, f.Size)
    }
    
    // Search
    fmt.Println("\n=== ค้นหา *.go ===")
    goFiles, _ := fm.Search("*.go")
    for _, f := range goFiles {
        fmt.Printf("  %s\n", f.Path)
    }
    
    // Write
    fm.Write("config/app.txt", "port=8080\ndebug=true\n")
    fmt.Println("\nสร้าง config/app.txt สำเร็จ")
    
    // Read
    content, _ := fm.Read("config/app.txt")
    fmt.Printf("เนื้อหา config:\n%s", content)
    
    // Stats
    fmt.Println("=== สถิติ ===")
    stats := fm.Stats()
    fmt.Printf("ไฟล์ทั้งหมด: %d\n", stats["total_files"])
    fmt.Printf("Directories: %d\n", stats["total_dirs"])
    fmt.Printf("ขนาดรวม: %d bytes\n", stats["total_size"])
    fmt.Println("Extensions:")
    for ext, count := range stats["extensions"].(map[string]int) {
        fmt.Printf("  %s: %d ไฟล์\n", ext, count)
    }
    
    // Export CSV
    fm.ExportToCSV("workspace.csv")
    fmt.Println("\nExport เป็น CSV สำเร็จ!")
    
    // แสดง CSV
    csvContent, _ := os.ReadFile("workspace.csv")
    scanner := bufio.NewScanner(strings.NewReader(string(csvContent)))
    lineCount := 0
    for scanner.Scan() && lineCount < 5 {
        fmt.Println(scanner.Text())
        lineCount++
    }
}
```

---

## สรุป

| ฟีเจอร์ | Package | ฟังก์ชันหลัก |
|---------|---------|------------|
| อ่าน/เขียนไฟล์ | `os` | `ReadFile`, `WriteFile`, `OpenFile` |
| Buffered I/O | `bufio` | `Scanner`, `Reader`, `Writer` |
| จัดการ Path | `filepath` | `Join`, `Dir`, `Base`, `Walk` |
| CSV | `encoding/csv` | `Reader`, `Writer` |

## Resources

- [os package documentation](https://pkg.go.dev/os)
- [bufio package documentation](https://pkg.go.dev/bufio)
- [filepath package documentation](https://pkg.go.dev/path/filepath)
- [encoding/csv documentation](https://pkg.go.dev/encoding/csv)
- [Go File I/O Tutorial](https://gobyexample.com/reading-files)
