
```c++
#include <mutex>

#if !defined(__clang__)
#pragma error "Thread safety analysis warning not support"
#endif

struct __attribute__((capability("mutex"))) SafeMutex {
	std::mutex mu;
	
	void lock() __attribute__((acquire_capability))
	{
		mu.lock();
	}
	
	void unlock() __attribute__((release_capability))
	{
		mu.unlock();
	}

};

struct BankAccount {
	SafeMutex mu;
	/// можно для нескольких членов класса указать к какому
	/// guard они связаны
	int balance __attribute__((guarded_by(mu)));
	
	BankAccount() : balance(0) {}
	
	void deposit(int amount) {
		/// Compile Ok
		mu.lock();
		balance += amount;
		mu.unlock();
	}
	
	int getBalance() {
		/// Compile Ok
		mu.lock();
		int result = balance;
		mu.unlock();
		return result;
	}
	
	int getBalanceUnsafeV2() {
		/// Compile error: mutex 'mu' is still held at the end of function
		mu.lock();
		auto cp{balance};
		//mu.unlock();
	}
	
	int getBalanceUnsafeV1() {
		/// Compile error: requires holding mutex 'mu'
		return balance;
	}
};

int main() {
	BankAccount acc;
	acc.deposit(100); // ✅ OK: сама захватывает
	int b = acc.getBalance(); // ✅ OK
	return 0;
}
```

```bash
clear && clang++ -std=c++11 -Wthread-safety -Werror main.cpp
```
