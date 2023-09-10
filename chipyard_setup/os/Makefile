SHELL = bash
CC = riscv64-unknown-linux-gnu-gcc
CFLAGS = -std=c99 -fno-common -fno-builtin-printf -static -Wall

SOURCE = program_name.c
TARGET = target_name # the name of .riscv file

all: $(TARGET)

$(TARGET): $(SOURCE)
	$(CC) $(CFLAGS) -o $(TARGET) $(SOURCE) -lpthread

.PHONY: clean
clean:
	rm -f $(TARGET)
