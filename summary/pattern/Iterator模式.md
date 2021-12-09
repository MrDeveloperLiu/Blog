#### Iterator模式

Iterator模式用于在数据集合中按照顺序遍历集合

* 定义协议`Iterator`
* 定义一个书架`BookShelf`，能够保存一些书籍`BookShelf.Book`
* 书架可以放置书籍，并可以产生一个迭代器`makeIterator()`

``` swift
protocol Iterator {
    associatedtype T
    mutating func next() -> Self.T?
}

struct BookShelf {
    
    struct Book {
        var name: String
    }

    struct BookIterator: Iterator {
        private var shelf: BookShelf
        
        init(_ shelf: BookShelf) {
            self.shelf = shelf
        }
        
        mutating func next() -> Book? {
            guard let b = shelf.books.first else {
                return nil
            }
            shelf.books.remove(at: 0)
            return b
        }
    }

    private(set) var books: [Book] = []
    
    mutating func put(_ b: Book) {
        books.append(b)
    }
    
    func makeIterator() -> BookIterator {
        return BookIterator(self)
    }
}
```
* 使用迭代器，遍历书架中的书
  
``` swift
var shelf = BookShelf()
shelf.put(BookShelf.Book(name: "Harry Potter"))
shelf.put(BookShelf.Book(name: "Green"))
shelf.put(BookShelf.Book(name: "Mary"))
 
var it = shelf.makeIterator()
while let book = it.next() {
    print("book \(book.name)")
}
```