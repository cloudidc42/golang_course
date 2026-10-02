# Part 92: Machine Learning ด้วย Go

## เป้าหมายของบทเรียน
- ML basics ใน Go
- gonum สำหรับ numerical computing
- Linear regression จาก scratch
- Neural network basics
- ONNX model inference
- Feature engineering

---

## 1. Numerical Computing ด้วย gonum

```go
// gonum_basics.go - gonum numerical computing
package main

import (
    "fmt"
    "math"
    "sort"
)

// Vector operations (ไม่ใช้ gonum เพื่อ portability)
type Vector []float64

func NewVector(values ...float64) Vector {
    return Vector(values)
}

func (v Vector) Add(other Vector) Vector {
    result := make(Vector, len(v))
    for i := range v {
        result[i] = v[i] + other[i]
    }
    return result
}

func (v Vector) Scale(s float64) Vector {
    result := make(Vector, len(v))
    for i := range v {
        result[i] = v[i] * s
    }
    return result
}

func (v Vector) Dot(other Vector) float64 {
    sum := 0.0
    for i := range v {
        sum += v[i] * other[i]
    }
    return sum
}

func (v Vector) Norm() float64 {
    return math.Sqrt(v.Dot(v))
}

func (v Vector) Mean() float64 {
    sum := 0.0
    for _, x := range v {
        sum += x
    }
    return sum / float64(len(v))
}

func (v Vector) Std() float64 {
    mean := v.Mean()
    sum := 0.0
    for _, x := range v {
        diff := x - mean
        sum += diff * diff
    }
    return math.Sqrt(sum / float64(len(v)))
}

// Matrix operations
type Matrix struct {
    rows, cols int
    data       []float64
}

func NewMatrix(rows, cols int) *Matrix {
    return &Matrix{
        rows: rows,
        cols: cols,
        data: make([]float64, rows*cols),
    }
}

func (m *Matrix) Set(i, j int, val float64) {
    m.data[i*m.cols+j] = val
}

func (m *Matrix) Get(i, j int) float64 {
    return m.data[i*m.cols+j]
}

func (m *Matrix) Mul(other *Matrix) *Matrix {
    if m.cols != other.rows {
        panic("incompatible dimensions")
    }
    
    result := NewMatrix(m.rows, other.cols)
    for i := 0; i < m.rows; i++ {
        for j := 0; j < other.cols; j++ {
            sum := 0.0
            for k := 0; k < m.cols; k++ {
                sum += m.Get(i, k) * other.Get(k, j)
            }
            result.Set(i, j, sum)
        }
    }
    return result
}

func (m *Matrix) Transpose() *Matrix {
    result := NewMatrix(m.cols, m.rows)
    for i := 0; i < m.rows; i++ {
        for j := 0; j < m.cols; j++ {
            result.Set(j, i, m.Get(i, j))
        }
    }
    return result
}

// Statistics functions
func Percentile(data []float64, p float64) float64 {
    sorted := make([]float64, len(data))
    copy(sorted, data)
    sort.Float64s(sorted)
    
    index := p / 100 * float64(len(sorted)-1)
    lower := int(index)
    upper := lower + 1
    
    if upper >= len(sorted) {
        return sorted[lower]
    }
    
    frac := index - float64(lower)
    return sorted[lower]*(1-frac) + sorted[upper]*frac
}

func main() {
    fmt.Println("=== Numerical Computing Demo ===\n")
    
    // Vector operations
    v1 := NewVector(1, 2, 3, 4, 5)
    v2 := NewVector(5, 4, 3, 2, 1)
    
    fmt.Printf("v1: %v\n", v1)
    fmt.Printf("v2: %v\n", v2)
    fmt.Printf("v1 + v2: %v\n", v1.Add(v2))
    fmt.Printf("v1 dot v2: %.2f\n", v1.Dot(v2))
    fmt.Printf("v1 norm: %.4f\n", v1.Norm())
    fmt.Printf("v1 mean: %.2f\n", v1.Mean())
    fmt.Printf("v1 std: %.4f\n", v1.Std())
    
    // Statistics
    data := []float64{23, 45, 12, 67, 34, 89, 56, 78, 90, 11}
    fmt.Printf("\nData: %v\n", data)
    fmt.Printf("P25: %.2f\n", Percentile(data, 25))
    fmt.Printf("P50: %.2f\n", Percentile(data, 50))
    fmt.Printf("P75: %.2f\n", Percentile(data, 75))
    fmt.Printf("P90: %.2f\n", Percentile(data, 90))
    
    // Matrix
    m1 := NewMatrix(2, 3)
    m1.Set(0, 0, 1); m1.Set(0, 1, 2); m1.Set(0, 2, 3)
    m1.Set(1, 0, 4); m1.Set(1, 1, 5); m1.Set(1, 2, 6)
    
    m2 := NewMatrix(3, 2)
    m2.Set(0, 0, 7); m2.Set(0, 1, 8)
    m2.Set(1, 0, 9); m2.Set(1, 1, 10)
    m2.Set(2, 0, 11); m2.Set(2, 1, 12)
    
    result := m1.Mul(m2)
    fmt.Printf("\nMatrix multiplication (2x3) * (3x2):\n")
    for i := 0; i < result.rows; i++ {
        for j := 0; j < result.cols; j++ {
            fmt.Printf("  [%d][%d] = %.0f\n", i, j, result.Get(i, j))
        }
    }
}
```

---

## 2. Linear Regression

```go
// linear_regression.go - Linear regression จาก scratch
package main

import (
    "fmt"
    "math"
    "math/rand"
)

// LinearRegression model
type LinearRegression struct {
    Weights  []float64
    Bias     float64
    LearningRate float64
    Epochs   int
}

// NewLinearRegression สร้าง model
func NewLinearRegression(features int, lr float64, epochs int) *LinearRegression {
    weights := make([]float64, features)
    for i := range weights {
        weights[i] = rand.NormFloat64() * 0.01
    }
    
    return &LinearRegression{
        Weights:      weights,
        Bias:         0,
        LearningRate: lr,
        Epochs:       epochs,
    }
}

// Predict ทำนาย output
func (lr *LinearRegression) Predict(x []float64) float64 {
    result := lr.Bias
    for i, w := range lr.Weights {
        result += w * x[i]
    }
    return result
}

// MSE คำนวณ Mean Squared Error
func (lr *LinearRegression) MSE(X [][]float64, y []float64) float64 {
    sum := 0.0
    for i, x := range X {
        pred := lr.Predict(x)
        diff := pred - y[i]
        sum += diff * diff
    }
    return sum / float64(len(y))
}

// Train train model ด้วย gradient descent
func (lr *LinearRegression) Train(X [][]float64, y []float64) []float64 {
    n := float64(len(y))
    history := make([]float64, 0, lr.Epochs)
    
    for epoch := 0; epoch < lr.Epochs; epoch++ {
        // คำนวณ gradients
        dw := make([]float64, len(lr.Weights))
        db := 0.0
        
        for i, x := range X {
            pred := lr.Predict(x)
            err := pred - y[i]
            
            for j, xj := range x {
                dw[j] += err * xj
            }
            db += err
        }
        
        // Update weights
        for j := range lr.Weights {
            lr.Weights[j] -= lr.LearningRate * dw[j] / n
        }
        lr.Bias -= lr.LearningRate * db / n
        
        loss := lr.MSE(X, y)
        history = append(history, loss)
        
        if epoch%100 == 0 {
            fmt.Printf("Epoch %d: MSE=%.6f\n", epoch, loss)
        }
    }
    
    return history
}

// generateData สร้าง synthetic data
func generateData(n int) ([][]float64, []float64) {
    X := make([][]float64, n)
    y := make([]float64, n)
    
    // y = 2*x1 + 3*x2 + 1 + noise
    for i := range X {
        x1 := rand.Float64()*10 - 5
        x2 := rand.Float64()*10 - 5
        X[i] = []float64{x1, x2}
        y[i] = 2*x1 + 3*x2 + 1 + rand.NormFloat64()*0.5
    }
    
    return X, y
}

// R2Score คำนวณ R² score
func R2Score(yTrue, yPred []float64) float64 {
    mean := 0.0
    for _, v := range yTrue {
        mean += v
    }
    mean /= float64(len(yTrue))
    
    ssTot := 0.0
    ssRes := 0.0
    for i, v := range yTrue {
        ssTot += (v - mean) * (v - mean)
        ssRes += (v - yPred[i]) * (v - yPred[i])
    }
    
    if ssTot == 0 {
        return 1.0
    }
    return 1 - ssRes/ssTot
}

func main() {
    fmt.Println("=== Linear Regression Demo ===\n")
    
    rand.Seed(42)
    
    // Generate data
    X, y := generateData(200)
    
    // Split train/test
    trainSize := 160
    XTrain, yTrain := X[:trainSize], y[:trainSize]
    XTest, yTest := X[trainSize:], y[trainSize:]
    
    // Train model
    model := NewLinearRegression(2, 0.01, 500)
    
    fmt.Println("Training...")
    model.Train(XTrain, yTrain)
    
    fmt.Printf("\nLearned weights: %.4f, %.4f\n", model.Weights[0], model.Weights[1])
    fmt.Printf("Learned bias:    %.4f\n", model.Bias)
    fmt.Println("Expected: w=[2, 3], b=1")
    
    // Evaluate
    predictions := make([]float64, len(XTest))
    for i, x := range XTest {
        predictions[i] = model.Predict(x)
    }
    
    mse := model.MSE(XTest, yTest)
    r2 := R2Score(yTest, predictions)
    rmse := math.Sqrt(mse)
    
    fmt.Printf("\nTest Results:\n")
    fmt.Printf("  MSE:  %.6f\n", mse)
    fmt.Printf("  RMSE: %.6f\n", rmse)
    fmt.Printf("  R²:   %.6f\n", r2)
}
```

---

## 3. Feature Engineering

```go
// feature_engineering.go - Feature engineering
package main

import (
    "fmt"
    "math"
    "sort"
    "strings"
)

// FeatureScaler normalizes features
type FeatureScaler struct {
    means  []float64
    stds   []float64
    fitted bool
}

// Fit คำนวณ mean และ std
func (s *FeatureScaler) Fit(X [][]float64) {
    if len(X) == 0 {
        return
    }
    
    n := len(X[0])
    s.means = make([]float64, n)
    s.stds = make([]float64, n)
    
    // Calculate means
    for _, x := range X {
        for j, v := range x {
            s.means[j] += v
        }
    }
    for j := range s.means {
        s.means[j] /= float64(len(X))
    }
    
    // Calculate stds
    for _, x := range X {
        for j, v := range x {
            diff := v - s.means[j]
            s.stds[j] += diff * diff
        }
    }
    for j := range s.stds {
        s.stds[j] = math.Sqrt(s.stds[j] / float64(len(X)))
        if s.stds[j] == 0 {
            s.stds[j] = 1 // Avoid division by zero
        }
    }
    
    s.fitted = true
}

// Transform normalizes data
func (s *FeatureScaler) Transform(X [][]float64) [][]float64 {
    if !s.fitted {
        panic("must fit before transform")
    }
    
    result := make([][]float64, len(X))
    for i, x := range X {
        row := make([]float64, len(x))
        for j, v := range x {
            row[j] = (v - s.means[j]) / s.stds[j]
        }
        result[i] = row
    }
    return result
}

// LabelEncoder encodes categorical values
type LabelEncoder struct {
    classes map[string]int
    reverse []string
}

func NewLabelEncoder() *LabelEncoder {
    return &LabelEncoder{
        classes: make(map[string]int),
    }
}

func (e *LabelEncoder) Fit(labels []string) {
    unique := make(map[string]bool)
    for _, l := range labels {
        unique[l] = true
    }
    
    e.reverse = make([]string, 0, len(unique))
    for l := range unique {
        e.reverse = append(e.reverse, l)
    }
    sort.Strings(e.reverse)
    
    for i, l := range e.reverse {
        e.classes[l] = i
    }
}

func (e *LabelEncoder) Transform(labels []string) []int {
    result := make([]int, len(labels))
    for i, l := range labels {
        if idx, ok := e.classes[l]; ok {
            result[i] = idx
        } else {
            result[i] = -1 // Unknown
        }
    }
    return result
}

// OneHotEncoder สร้าง one-hot encoding
func OneHotEncode(categories []string) [][]float64 {
    encoder := NewLabelEncoder()
    encoder.Fit(categories)
    
    encoded := encoder.Transform(categories)
    n := len(encoder.classes)
    
    result := make([][]float64, len(categories))
    for i, idx := range encoded {
        row := make([]float64, n)
        if idx >= 0 {
            row[idx] = 1.0
        }
        result[i] = row
    }
    
    return result
}

// TF-IDF Text Vectorizer
type TFIDFVectorizer struct {
    vocab   map[string]int
    idf     []float64
    fitted  bool
}

func NewTFIDFVectorizer() *TFIDFVectorizer {
    return &TFIDFVectorizer{
        vocab: make(map[string]int),
    }
}

func tokenize(text string) []string {
    text = strings.ToLower(text)
    return strings.Fields(text)
}

func (v *TFIDFVectorizer) Fit(documents []string) {
    // Build vocabulary
    docFreq := make(map[string]int)
    
    for _, doc := range documents {
        words := tokenize(doc)
        seen := make(map[string]bool)
        
        for _, w := range words {
            v.vocab[w] = 0 // placeholder
            if !seen[w] {
                docFreq[w]++
                seen[w] = true
            }
        }
    }
    
    // Assign indices
    idx := 0
    for w := range v.vocab {
        v.vocab[w] = idx
        idx++
    }
    
    // Calculate IDF
    n := float64(len(documents))
    v.idf = make([]float64, len(v.vocab))
    for w, i := range v.vocab {
        df := float64(docFreq[w])
        v.idf[i] = math.Log(n/df) + 1
    }
    
    v.fitted = true
}

func (v *TFIDFVectorizer) Transform(documents []string) [][]float64 {
    result := make([][]float64, len(documents))
    
    for i, doc := range documents {
        vector := make([]float64, len(v.vocab))
        words := tokenize(doc)
        
        // TF
        tf := make(map[string]float64)
        for _, w := range words {
            tf[w]++
        }
        total := float64(len(words))
        
        // TF-IDF
        for w, count := range tf {
            if idx, ok := v.vocab[w]; ok {
                vector[idx] = (count / total) * v.idf[idx]
            }
        }
        
        result[i] = vector
    }
    
    return result
}

func main() {
    fmt.Println("=== Feature Engineering Demo ===\n")
    
    // Normalization
    fmt.Println("1. Feature Normalization")
    X := [][]float64{
        {1, 100, 0.5},
        {2, 200, 1.5},
        {3, 300, 2.5},
        {4, 400, 3.5},
    }
    
    scaler := &FeatureScaler{}
    scaler.Fit(X)
    Xnorm := scaler.Transform(X)
    
    fmt.Println("Original -> Normalized:")
    for i := range X {
        fmt.Printf("  %v -> [%.2f, %.2f, %.2f]\n",
            X[i], Xnorm[i][0], Xnorm[i][1], Xnorm[i][2])
    }
    
    // Label Encoding
    fmt.Println("\n2. Label Encoding")
    labels := []string{"cat", "dog", "bird", "cat", "dog"}
    encoder := NewLabelEncoder()
    encoder.Fit(labels)
    encoded := encoder.Transform(labels)
    fmt.Printf("Labels: %v\n", labels)
    fmt.Printf("Encoded: %v\n", encoded)
    
    // One-Hot Encoding
    fmt.Println("\n3. One-Hot Encoding")
    categories := []string{"red", "blue", "green", "red", "blue"}
    oneHot := OneHotEncode(categories)
    fmt.Printf("Categories: %v\n", categories)
    for i, cat := range categories {
        fmt.Printf("  %s -> %v\n", cat, oneHot[i])
    }
    
    // TF-IDF
    fmt.Println("\n4. TF-IDF Vectorization")
    docs := []string{
        "go is a programming language",
        "go is fast and efficient",
        "programming in go is fun",
    }
    
    tfidf := NewTFIDFVectorizer()
    tfidf.Fit(docs)
    vectors := tfidf.Transform(docs)
    
    fmt.Printf("Documents: %d, Vocabulary size: %d\n", len(docs), len(tfidf.vocab))
    fmt.Printf("Vector dimensions: %d\n", len(vectors[0]))
}
```

---

## สรุป

บทนี้ครอบคลุม Machine Learning ด้วย Go:

1. **Numerical Computing** - vectors, matrices, statistics
2. **Linear Regression** - gradient descent, MSE, R² score
3. **Feature Engineering** - scaling, encoding, TF-IDF
4. **gonum** - production numerical computing library
5. **ONNX** - รัน pre-trained models

### Key Takeaways

- Go เหมาะกับ ML inference (serving) มากกว่า training
- gonum ให้ numerical computing คล้าย numpy
- Feature engineering สำคัญมากสำหรับ model performance
- ONNX เป็น standard format สำหรับ model portability
- ใช้ Python สำหรับ training, Go สำหรับ serving เป็น pattern ที่ดี
