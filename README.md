# 說明
程式碼透過 Windows API 調用來取得並顯示實體硬碟的序號、型號以及韌體版本。

```
#pragma once
#define _CRT_SECURE_NO_WARNINGS
#define no_init_all
#include <Windows.h>
#include <iostream>
#include <iomanip>	// For std::setw

#define ErrorPrint(x) \
	do { \
		std::wcerr << "Error at line " << __LINE__ << ": " << x << " (Error code: " << GetLastError() << ")" << std::endl; \
	} while (0)

#define DRIVE					0		// Drive ID
#define IDENTIFY_BUFFER_SIZE			512
#define ID_CMD					0xEC	// Returns ID sector for ATA

typedef struct _IDENTIFY_DATA
{
	USHORT GeneralConfiguration;		// 00  00
	USHORT NumberOfCylinders;		// 02  1
	USHORT Reserved1;			// 04  2
	USHORT NumberOfHeads;			// 06  3
	USHORT UnformattedBytesPerTrack;	// 08  4
	USHORT UnformattedBytesPerSector;	// 0A  5
	USHORT SectorsPerTrack;			// 0C  6
	USHORT VendorUnique1[3];		// 0E  7-9
	USHORT SerialNumber[10];		// 14  10-19
	USHORT BufferType;			// 28  20
	USHORT BufferSectorSize;		// 2A  21
	USHORT NumberOfEccBytes;		// 2C  22
	USHORT FirmwareRevision[4];		// 2E  23-26
	USHORT ModelNumber[20];			// 36  27-46
	UCHAR MaximumBlockTransfer;		// 5E  47
	UCHAR VendorUnique2;			// 5F
	USHORT DoubleWordIo;			// 60  48
	USHORT Capabilities;			// 62  49
	USHORT Reserved2;			// 64  50
	UCHAR VendorUnique3;			// 66  51
	UCHAR PioCycleTimingMode;		// 67
	UCHAR VendorUnique4;			// 68  52
	UCHAR DmaCycleTimingMode;		// 69
	USHORT TranslationFieldsValid : 1;	// 6A  53
	USHORT Reserved3 : 15;
	USHORT NumberOfCurrentCylinders;	// 6C  54
	USHORT NumberOfCurrentHeads;		// 6E  55
	USHORT CurrentSectorsPerTrack;		// 70  56
	ULONG CurrentSectorCapacity;		// 72  57-58
	USHORT CurrentMultiSectorSetting;	//     59
	ULONG UserAddressableSectors;		//     60-61
	USHORT SingleWordDMASupport : 8;	//     62
	USHORT SingleWordDMAActive : 8;
	USHORT MultiWordDMASupport : 8;		//     63
	USHORT MultiWordDMAActive : 8;
	USHORT AdvancedPIOModes : 8;		//     64
	USHORT Reserved4 : 8;
	USHORT MinimumMWXferCycleTime;		//     65
	USHORT RecommendedMWXferCycleTime;	//     66
	USHORT MinimumPIOCycleTime;		//     67
	USHORT MinimumPIOCycleTimeIORDY;	//     68
	USHORT Reserved5[2];				//     69-70
	USHORT ReleaseTimeOverlapped;		//     71
	USHORT ReleaseTimeServiceCommand;	//     72
	USHORT MajorRevision;			//     73
	USHORT MinorRevision;			//     74
	USHORT Reserved6[50];			//     75-126
	USHORT SpecialFunctionsEnabled;		//     127
	USHORT Reserved7[128];			//     128-255
} IDENTIFY_DATA, *PIDENTIFY_DATA;

void PrintDriveInfo(int iDrive, DWORD diskData[256]);
char *ConvertToString(DWORD diskData[256], int iFirstIndex, int iLastIndex, char *pcszBuf);


int main(int argc, char *argv[])
{
	char driveName[MAX_PATH];
	sprintf(driveName, "\\\\.\\PhysicalDrive%d", DRIVE);

	// Get handle of the physical drive
	HANDLE hDrive = CreateFileA(driveName, GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ, 0, OPEN_EXISTING, 0, 0);

	// Check the validity of the file handle
	if (hDrive == INVALID_HANDLE_VALUE)
	{
		ErrorPrint(L"CreateFile");
		exit(1);
	}
	
	// Prepare the command buffer
	ULONG CommandSize = sizeof(SENDCMDINPARAMS) + IDENTIFY_BUFFER_SIZE;
	PSENDCMDINPARAMS Command = (PSENDCMDINPARAMS)malloc(CommandSize);
	// Prepare the command
	Command->irDriveRegs.bCommandReg = ID_CMD;

	DWORD bytesReturned = 0;
	// Retrieve IDENTIFY data
	if (DeviceIoControl(hDrive,
		SMART_RCV_DRIVE_DATA, // 0x7C088
		Command,
		sizeof(SENDCMDINPARAMS),
		Command,
		CommandSize,
		&bytesReturned,
		NULL))
	{
		// Print the IDENTIFY data
		DWORD diskData[256]; // 256 is the IDENTIFY_DATA structure's size
		USHORT *punIdSector = (USHORT *)(PIDENTIFY_DATA)((PSENDCMDOUTPARAMS)Command)->bBuffer;
		for (int i = 0; i < 256; i++)
			diskData[i] = punIdSector[i];

		PrintDriveInfo(0, diskData);
	}
	else ErrorPrint(L"DeviceIoControl (SMART_RCV_DRIVE_DATA)");

	CloseHandle(hDrive);
}

void PrintDriveInfo(int iDrive, DWORD diskData[256])
{
	char serialNumber[1024];
	char modelNumber[1024];
	char revisionNumber[1024];

	// Copy the hard drive serial, model, and revision number to respective buffer
	ConvertToString(diskData, 10, 19, serialNumber);
	ConvertToString(diskData, 27, 46, modelNumber);
	ConvertToString(diskData, 23, 26, revisionNumber);

	std::cout << "------------------------ [Drive " << DRIVE << "] ------------------------" << std::endl;
	std::cout << std::left << std::setw(35) << "Drive Serial Number________________ : [" << serialNumber << "]\n";
	std::cout << std::left << std::setw(35) << "Drive Model Number_________________ : [" << modelNumber << "]\n";
	std::cout << std::left << std::setw(35) << "Drive Controller Revision Number___ : [" << revisionNumber << "]\n";
	
	std::cout << "Drive Type_________________________ : [";
	if (diskData[0] & 0x0080)
		std::cout << "Removable]\n";
	else if (diskData[0] & 0x0040)
		std::cout << "Fixed]\n";
	else
		std::cout << "Unknown]\n";
}

char *ConvertToString(DWORD diskData[256], int iFirstIndex, int iLastIndex, char *pcszBuf)
{
	int index = 0;
	int position = 0;

	// NOTE: each integer has two characters stored in it backwards
	for (index = iFirstIndex; index <= iLastIndex; index++)
	{
		// Get high byte for 1st character
		pcszBuf[position++] = (char)(diskData[index] / 256); // / 256 = >> 8
		// Get low byte for 2nd character
		pcszBuf[position++] = (char)(diskData[index] % 256); // ^ 256 = & 0xFF
	}

	// End the string and cut off trailing blanks
	pcszBuf[position] = '\0';
	for (index = position - 1; index > 0 && isspace(pcszBuf[index]); index--)
		pcszBuf[index] = '\0';

	return pcszBuf;
}
```
