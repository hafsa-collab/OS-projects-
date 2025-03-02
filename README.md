/* Description: A multi-threaded utility class that prints characters of a string in round-robin order across specified threads.*/

#include <iostream>
#include <string>
#include <mutex>
#include <vector>
#include <condition_variable>
#include <thread>
#include <chrono>

using namespace std;

class MyPrinter {
private:
   string str;         // Input string to be printed
   int char_count;     // Number of characters each thread prints per turn
   int thread_count;   // Total number of threads participating
   vector<thread> threads;       // Stores thread objects
   vector<std::thread::id> thread_ids;  // Maps system thread IDs to logical indices
   int thread_id;      // Unused in current implementation
   int allowed_thread; // Index of the thread allowed to print next (0-based)
   mutex mutex_lock;   // Mutex for synchronization
   condition_variable cv;  // Condition variable for thread coordination
   int next_char;      // Position of next character to print in the string

public:
    // Constructor initializes core parameters
    MyPrinter(string s, int c_count, int t_count) {
        str = s;
        char_count = c_count;
        thread_count = t_count;
        thread_id = 0;          // Currently unused
        next_char = 0;          // Start from beginning of string
        allowed_thread = 0;     // First thread to print is index 0
    }

    // Converts system thread ID to our logical thread index (0-based)
    int getCurrentThreadId(const std::thread::id& id) {
        int thread_id = 0;
        for(auto& e : thread_ids) {
            if(e == id) return thread_id;
            thread_id++;
        }
        return -1;  // Indicates invalid thread ID
    }

    // Main method to start and manage threads
    void run() {
        // Create and launch all threads
        for (int i = 0; i < thread_count; i++) {
            thread t(&MyPrinter::print_thread, this);
            cout << "Thread " << t.get_id() <<  " is " << i << endl;
            thread_ids.push_back(t.get_id());
            threads.push_back(move(t));
        }

        // Wait for all threads to complete execution
        for (int i = 0; i < thread_count; i++){
            threads[i].join();
        }
    }

    // Busy-wait until all threads are initialized (caution: CPU intensive)
    void waitforallthreadinit() {
        while(1) {
            if(thread_count == thread_ids.size()) return;
        }
    }

    // Thread execution function managing print synchronization
    void print_thread() {
        while(1) {
            waitforallthreadinit();  // Ensure all threads are ready
            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
            
            // Lock mutex and wait for turn using condition variable
            unique_lock<mutex> lock(mutex_lock);
            cv.wait(lock, [this] { 
                // Only wake if current thread is next in allowed_thread order
                return std::this_thread::get_id() == thread_ids[allowed_thread]; 
            });
            
            print_chars();  // Perform actual character printing
            
            // Update allowed_thread index with wrap-around
            allowed_thread++;
            if(allowed_thread == thread_count) allowed_thread = 0;
            
            // Adjust next_char position if exceeding string length
            if(next_char >= str.length()) next_char -= str.length();
            
            lock.unlock();
            cv.notify_all();  // Alert other threads to check their turn
        }
    }

    // Prints assigned characters and updates next_char position
    void print_chars() {
        cout << "ThreadId " << getCurrentThreadId(std::this_thread::get_id()) << " : ";
        int printcount = 0;
        
        // Print characters from current position to end of string
        for(int i=next_char; i < str.length() && printcount < char_count; i++){
            cout << str[i];
            printcount++;
        }
        
        // If needed, wrap around to beginning of string
        if(printcount < char_count) {
            for(int i=0; i<char_count - printcount; i++) {
                cout << str[i];
            }
        }
        
        // Update next_char position (may exceed string length temporarily)
        next_char = next_char + char_count;
        cout << endl;
    }
};

int main(int argc, char *argv[]) {
    // Validate command line arguments
    if (argc != 4) {
        cout << "Please provide 3 arguments - a string, char count & thread count" << endl;
        return 1;
    }

    // Parse input parameters
    string str = argv[1];
    int char_count = atoi(argv[2]);
    int thread_count = atoi(argv[3]);
    
    // Create printer instance and start processing
    MyPrinter p(str, char_count, thread_count);
    p.run();

    return 0;
}
