```c++
#include <stdio.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <net/if.h>
#include <linux/if_link.h>

int main(int argc, char **argv) {
	struct bpf_object *obj;
	int prog_fd, ifindex;
	const char *ifname = "en1";
	const char *xdp_obj_path = "xdp.o";
	const char *xdp_app_name = "xdp_main";
	
	// 1. Получаем индекс интерфейса
	ifindex = if_nametoindex(ifname);
	if (!ifindex) {/*handle error*/}
	
	// 2. Загружаем объект BPF
	obj = bpf_object__open(xdp_obj_path);
	if (libbpf_get_error(obj)) {/*handle error*/}
	
	// 3. Загружаем программу в ядро
	if (bpf_object__load(obj)) {/*handle error*/}
	
	// 4. Получаем файловый дескриптор программы
	prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, xdp_app_name));
	if (prog_fd < 0) {/*handle error*/}
	
	// 5. Прикрепляем XDP-программу к интерфейсу (в режиме DRV/NATIVE)
	unsigned int flags = XDP_FLAGS_UPDATE_IF_NOEXIST;
	if (bpf_set_link_xdp_fd(ifindex, prog_fd, flags) < 0) {/*handle error*/}
	
	// do something ...
	  
	// 6. Отсоединяем программу
	bpf_set_link_xdp_fd(ifindex, -1, flags);
	bpf_object__close(obj);
	return 0;
}
```

### CMake для сборки
```cmake
cmake_minimum_required(VERSION 3.14)
SET(UNIT_NAME xdp_hello_world)

project(${UNIT_NAME})

add_executable(${UNIT_NAME} ${UNIT_NAME}.cpp)

target_include_directories(${UNIT_NAME} PUBLIC /usr/include/libbpf)

target_link_libraries(${UNIT_NAME}
PRIVATE
bpf
elf
z
)
```
