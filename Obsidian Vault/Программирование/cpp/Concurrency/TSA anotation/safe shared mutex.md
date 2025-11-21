```c++
#include <mutex>
#include <string>

#if !defined(__clang__)
#pragma error "Thread safety analysis warning not support"
#endif

// Мьютекс с атрибутом capability
struct __attribute__((capability("mutex"))) RWLock {
	void lock() __attribute__((acquire_capability)) {}
	void unlock() __attribute__((release_capability)) {}
	void lock_shared() __attribute__((acquire_shared_capability)) {}
	void unlock_shared() __attribute__((release_shared_capability)) {}
};

struct Database {
	mutable RWLock m;
	/// можно для нескольких членов класса указать к какому
	/// guard они связаны
	std::string data __attribute__((guarded_by(m)));
	
	// Запись требует EXCLUSIVE блокировки
	void write(const std::string& s) {
		/// Compile ok
		m.lock();
		data = s;
		m.unlock();
	}
	
	// Чтение требует SHARED блокировки
	std::string read() const {
		/// Compile ok
		m.lock_shared();
		auto value = data;
		m.unlock_shared();
		return value;
	}
	
	void unsafeWrite(std::string s) {
		/// Compile error: writing variable 'data' requires holding 
		/// mutex 'm' exclusively
		data = s;
	}
	
	std::string usafeRead() const {
		/// Compile error: reading variable 'data' requires holding mutex 'm'
		return data;
	}
};
```

```bash
clear && clang++ -std=c++11 -Wthread-safety -Werror main.cpp
```