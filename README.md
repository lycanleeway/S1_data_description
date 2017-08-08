# S1_data_description
Novatel_S1_Data_Description

Only  RAWIMU   (other data structure needs addition)
2017.08.08
// DataConvert.cpp : Defines the entry point for the console application.
//

#include "stdafx.h"
#include<stdio.h>
#include<Windows.h>
#include <iostream>
#include<string.h>



struct BinaryMessage                                         //// reference Table 4
{
	unsigned char    HeaderLgth;
	unsigned short   MessageID;
	char             MessageType;
	unsigned char    PortAddress;
	unsigned short   MessageLgth;
	unsigned short   Sequence;
	unsigned char    IdleTime;
	unsigned char    TimeStatus;      //Enum
	unsigned short   Week;
	unsigned long    GPSms;           //GPSec
	unsigned long    ReceiverStatus;
	unsigned short   Reversed;       // internal use
	unsigned short   ReceiverVersion;  //internal  use

};

struct ShortBinaryMessage                                  /// reference  Table 7
{
	unsigned char	MessageLgth;
	unsigned short  MessageID;
	unsigned short  GNSSWeek;
	unsigned long   GNSSms;
	//long            GNSSms;
};

struct RAWIMUS                                             ///  reference   P179
{
	unsigned long  GNSSWeek;
	double         SecsintoWeek;
	long           IMUStatus;
	long           Z;
	long	       Y;/////-y
	long           X;
	long           ZG;
	long           YG;////-yg
	long           XG;
};

struct BESTGNSSPOS
{
	long      SOL_Status;  //Enum
	long      POS_Type;    //Enum
	double    Lat;
	double    Lon;
	double    Hgt;
	float     Undulation;
	long      DatumID;    //Enum
	float     Latdev;
	float     Londev;
	float     Hgtdev;
	long      StnID;      //char[4]
	float     Diff_age;
	float     Sol_age;
	unsigned char  SV;

};



int main()
{
	FILE *fp;
	if ((fp = fopen("new.gps", "r")) == NULL)  //读取文件
	{
		printf("读取文件失败 \n ");
		exit(1);
	}
	std::cout << "读取" << "成功" << std::endl;

	long size;
	fseek(fp, 0, SEEK_END);   ///将文件指针移动文件结尾
	size = ftell(fp); ///求出当前文件指针距离文件开始的字节数
	printf("Length of the GPS file: %ld Bytes.\n", size);
  
	
	
	fseek(fp, 0, SEEK_SET);   //文件起始位置
//	while (!feof(fp))
//	{
//		printf("%2x \n", unsigned char(fgetc(fp)));//每次获取一个字符并打印
//	}

/*	char buffer[2];
//	fgets(buffer,2, fp);
	printf("%2x\n",buffer);
	if (buffer == "AA44")
		printf("header");
	else
		printf("no starting");
*/
///// 数据读入数组////////////////
	//printf("%d\n", int(size));

	unsigned char *s = (unsigned char*)malloc(sizeof(char) * size);
	for (int i = 0; i <int(size); i++)             //int(size)
	{
		fseek(fp, i, SEEK_SET);
		s[i] = fgetc(fp);
		printf("%d\n", i);
	}

	fclose(fp);
//	if (int(s[18]) == 176)
//	    printf("ok");
//	else
//		printf("no");

//	printf("%x\n", s[18]);
//	printf("%x\n", s[19]);               ////******** 此处数据读取有问题 第19位（从0开始），输出应为1A，实际却是结束符 ff   1A有问题
//	printf("%x\n", s[20]);

	FILE *fout;
	fout = fopen("out.txt","w");
	fprintf(fout, "注： IMUStatus中因为使用unsigned char 故 2a000 等价于2a000000,即0等价于00 \n");
	fprintf(fout, "\n");
//	fprintf(fout, "aaa");

	for (int k = 0; k < int(size); k++)                 //int(size)
	{
		if ((s[k] == 0xAA) && (s[k + 1] == 0x44) && (s[k + 2] == 0x12))
		{
			BinaryMessage B;
			//printf("BinaryMessage\n");
			B.MessageID = int(s[k + 5]) * 256 + int(s[k + 4]);                /// reverse
			switch (B.MessageID)
		  {
			 case 1429:
			 {
				 fprintf(fout, "%#BESTGNSSPOSA; \n");
				break;
			 }
			 case 42:
			 {
				 fprintf(fout, "%#BESTPOSA; \n");
				 break;
			 }
			 case 1430:
			 {
				 fprintf(fout, "%#BESTGNSSVELA; \n");
				 break;
			 }
			 default:
				break;
		  }

		}

		else if ((s[k] == 0xAA) && (s[k + 1] == 0x44) && (s[k + 2] == 0x13))
		{
			ShortBinaryMessage SB;
			//fprintf(fout, "%%");
			SB.MessageID = int(s[k + 5])*256 + int(s[k + 4]);                /// reverse
			switch (SB.MessageID)
		  {
			case 325:
			{
				fprintf(fout, "%%RAWIMUSA,");
				SB.GNSSWeek = int(s[k + 7]) * 256 + int(s[k + 6]);             // reverse
				fprintf(fout, "%d,", SB.GNSSWeek);
				//reverse    1a总被ff替换 这里直接写1a  间隔7 or 9  Convert4软件  通过奇偶判读 间隔 +1
				SB.GNSSms =  int(/*s[k+11]*/0x1a)*256*256*256 + int(s[k+10])*256*256 + int(s[k+9])*256 + int(s[k+8]);   
				fprintf(fout, "%f ms ;", double(SB.GNSSms)/1000); // float 除以1000的标度

				RAWIMUS IMU;
				IMU.GNSSWeek = int(s[k + 15]) * 256 * 256 * 256 + int(s[k + 14]) * 256 * 256 + int(s[k + 13]) * 256 + int(s[k + 12]);        //reverse
				fprintf(fout, "%d,", IMU.GNSSWeek);
				IMU.SecsintoWeek =   SB.GNSSms ; ///  这里的精细时间计算有问题 8bytes 暂时用header中的时间
				fprintf(fout, "%f ms ;", double(IMU.SecsintoWeek) / 1000); // float 除以1000的标度
				fprintf(fout, "%x%x%x%x,", s[k + 27], s[k + 26], s[k + 25], s[k + 24]);    //IMUStatus
				////zyx accel
				IMU.Z= int(s[k + 31]) * 256 * 256 * 256 + int(s[k + 30]) * 256 * 256 + int(s[k + 29]) * 256 + int(s[k + 28]);
				fprintf(fout, "%d,", IMU.Z);
				IMU.Y = int(s[k + 35]) * 256 * 256 * 256 + int(s[k + 34]) * 256 * 256 + int(s[k + 33]) * 256 + int(s[k + 32]);
				fprintf(fout, "%d,", IMU.Y);
				IMU.X = (int(s[k + 39]) * 256 * 256 * 256 + int(s[k + 38]) * 256 * 256 + int(s[k + 37]) * 256 + int(s[k + 36]))-4294967295-1;
				fprintf(fout, "%d,", IMU.X);
				////zyx gyro
				IMU.ZG = int(s[k + 43]) * 256 * 256 * 256 + int(s[k + 42]) * 256 * 256 + int(s[k + 41]) * 256 + int(s[k + 40]);
				fprintf(fout, "%d,", IMU.ZG);
				IMU.YG =  (int(s[k + 47]) * 256 * 256 * 256 + int(s[k + 46]) * 256 * 256 + int(s[k + 45]) * 256 + int(s[k + 44])) - 1-4294967295;
				fprintf(fout, "%d,", IMU.YG);
				IMU.XG = (int(s[k + 51]) * 256 * 256 * 256 + int(s[k + 50]) * 256 * 256 + int(s[k + 49]) * 256 + int(s[k + 48])) - 1 - 4294967295;
				fprintf(fout, "%d", IMU.XG);
				fprintf(fout, "\n");
				break;
			}
			case 508:
			{
				fprintf(fout, "%%INSPVASA; \n");
				break;
			}
			default:
				break;
		  }
         
	
		}
	}

	return 0;
}

