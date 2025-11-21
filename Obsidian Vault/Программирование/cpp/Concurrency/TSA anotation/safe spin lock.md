```c++
#include <atomic>
#include <thread>
  
// Спин-лок (busy-wait lock)
class __attribute__((capability("spinlock"))) SpinLock {
	std::atomic<bool> locked_ = {false};
public:

	// Захват с аннотацией
	void lock() __attribute__((acquire_capability)) {
		// Spin-wait
		while (locked_.exchange(true, std::memory_order_acquire)) {
			std::this_thread::yield(); // Даем другим потокам шанс
		}
	}
	
	// Попытка захвата
	bool try_lock() __attribute__((try_acquire_capability(true))) {
		return !locked_.exchange(true, std::memory_order_acquire);
	}
	
	// Освобождение
	void unlock() __attribute__((release_capability)) {
		locked_.store(false, std::memory_order_release);
	}
};

// Использование
class HighPerformanceBuffer {
	SpinLock spinlock_;
	int buffer_[1024] __attribute__((guarded_by(spinlock_)));
	size_t size_ __attribute__((guarded_by(spinlock_)));
public:
	
	void add(int value) {
		spinlock_.lock(); // Короткий захват
		if (size_ < 1024) {
			buffer_[size_++] = value;
		}
		spinlock_.unlock();
	}
	
	// Метод для lock-free чтения (без блокировки)
	bool try_get(int& out) {
		if (spinlock_.try_lock()) { // Пробуем захватить
			if (size_ > 0) {
				out = buffer_[size_ - 1];
	            spinlock_.unlock();
	            return true;
			}
			spinlock_.unlock();
		}
		return false;
	}

};
```