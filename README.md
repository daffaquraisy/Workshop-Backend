# Setup

- run this following script `npm i -g @nestjs/cli`
- run this following script `nest new project-name`
- run this following script `npm handlebars pg puppeteer puppeteer-report @pgtyped/runtime @nestjs/config dotenv`
- run this following script `npm i -D @pgtyped/cli typescript`
- run this following script `nest g resource report --no-spec`
- add the following script inside the `package.json` file: `"start:pg": "npx pgtyped -w -c config.json"`
- create a `config.json` file in the root folder
- add the following script inside the `config.json` file:

```json
{
  "transforms": [
    {
      "mode": "ts",
      "include": "**/*.ts",
      "emitTemplate": "{{dir}}/{{name}}.types.ts"
    }
  ],
  "typesOverrides": {
    "date": "Date",
    "timestamptz": "string",
    "numeric": "number",
    "int8": "BigInt"
  },
  "srcDir": "./src/",
  "dbUrl": "postgresql://username:password@ip:port/dbname"
}
```

- remove `app.service.ts`, `app.controller.ts`, and `app.controller.spec.ts` file
- run this following script `nest g resource database --no-spec`
- add the following syntax for `database.service.ts` :

```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Pool, PoolConfig, QueryResult } from 'pg';

@Injectable()
export class DatabaseService {
  private pool: Pool;

  constructor(private configService: ConfigService) {
    const poolConfig: PoolConfig = {
      user: this.configService.get('DB_USER'),
      host: this.configService.get('DB_HOST'),
      database: this.configService.get('DB_DATABASE'),
      password: this.configService.get('DB_PASSWORD'),
      port: this.configService.get('DB_PORT'),
      max: 20,
    };

    this.pool = new Pool(poolConfig);
  }

  async query(text: string, params?: any[]): Promise<QueryResult> {
    return this.pool.query(text, params);
  }

  end() {
    this.pool.end();
  }
}
```

- add the following syntax for `database.module.ts` :

```typescript
import { Global, Module } from '@nestjs/common';
import { DatabaseService } from './database.service';

@Global()
@Module({
  providers: [DatabaseService],
  exports: [DatabaseService],
})
export class DatabaseModule {}
```

- add the following syntax for `app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    DatabaseModule,
    ConfigModule.forRoot({
      isGlobal: true,
    }),
  ],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

- open two terminal tabs. In one, run `npm run start:pg`, and in the other, run `npm run start:dev`

## Prepare Helper

- create a folder named "utils", then create a file named `generate.pdf.ts` inside the "utils" folder
- add the following syntax for the `generate.pdf.ts` :

```typescript
const report = require('puppeteer-report');
const puppeteer = require('puppeteer');
const hbs = require('handlebars');
const fs = require('fs-extra');

export const createPdf = async (filePath, options = {}, data = {}) => {
  try {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();

    hbs.registerHelper('ifCond', function (v1, operator, v2, options) {
      switch (operator) {
        case '==':
          return v1 == v2 ? options.fn(data) : options.inverse(data);
        case '===':
          return v1 === v2 ? options.fn(data) : options.inverse(data);
        case '!=':
          return v1 != v2 ? options.fn(data) : options.inverse(data);
        case '!==':
          return v1 !== v2 ? options.fn(data) : options.inverse(data);
        case '<':
          return v1 < v2 ? options.fn(data) : options.inverse(data);
        case '<=':
          return v1 <= v2 ? options.fn(data) : options.inverse(data);
        case '>':
          return v1 > v2 ? options.fn(data) : options.inverse(data);
        case '>=':
          return v1 >= v2 ? options.fn(data) : options.inverse(data);
        case '&&':
          return v1 && v2 ? options.fn(data) : options.inverse(data);
        case '||':
          return v1 || v2 ? options.fn(data) : options.inverse(data);
        default:
          return options.inverse(options);
      }
    });

    hbs.registerHelper({
      eq: (v1, v2) => v1 === v2,
      ne: (v1, v2) => v1 !== v2,
      lt: (v1, v2) => v1 < v2,
      gt: (v1, v2) => v1 > v2,
      lte: (v1, v2) => v1 <= v2,
      gte: (v1, v2) => v1 >= v2,
      and() {
        return Array.prototype.every.call(arguments, Boolean);
      },
      or() {
        return Array.prototype.slice.call(arguments, 0, -1).some(Boolean);
      },
    });

    const html = await fs.readFile(filePath, 'utf8');
    const content = hbs.compile(html)(data);

    await page.setContent(content);

    const pdfOptions = {
      format: 'A4',
      printBackground: true,
      displayHeaderFooter: true,
      margin: {
        top: '0.8in',
        bottom: '0.6in',
        left: '1in',
        right: '0.6in',
      },
      headerTemplate: `<p></p>`,
      footerTemplate: `
			<div style="width: 100%; text-align: center; font-size: 10px;">
					<strong>RAHASIA</strong>
					<br>
			</div>`,
      ...options,
    };

    const buffer = await report.pdfPage(page, pdfOptions);
    await browser.close();
    return buffer;
  } catch (e) {
    console.log(e);
  }
};
```

## Prepare Query

- Execute this query to create a table and insert data into it.

```sql
CREATE TABLE pesawat (
    id serial primary key not null ,
    regNumber VARCHAR(255) UNIQUE,
    lanud VARCHAR(255),
    koopsud VARCHAR(255)
);

INSERT INTO pesawat (regNumber, lanud, koopsud) VALUES
('A-1234', 'Hlm', 'Koopsud I'),
('A-5678', 'Adi', 'Kodiklatau'),
('A-9012', 'Ats', 'Koopsud I'),
('A-3456', 'Iwj', 'Koopsud II'),
('A-7890', 'Hlm', 'Koopsud I'),
('A-2345', 'Adi', 'Kodiklatau'),
('A-6789', 'Ats', 'Koopsud I'),
('A-0123', 'Iwj', 'Koopsud II'),
('A-4567', 'Hlm', 'Koopsud I'),
('A-8901', 'Adi', 'Kodiklatau');
```

- create a file named `report/report.queries.ts` inside folder `report` and add this following script :

```typescript
import { sql } from '@pgtyped/runtime';
import { ISelectPesawatQuery } from './report.queries.types';

export const selectPesawat = sql<ISelectPesawatQuery>`
  SELECT
    ROW_NUMBER() OVER (ORDER BY regNumber) AS no,
    regNumber,
    lanud,
    koopsud
  FROM
    pesawat
`;
```

## Prepare Repository

- create a file named `report/report.repository.ts` inside folder `report` and add this following script :

```typescript
import { Injectable } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import { selectPesawat } from './report.queries';

@Injectable()
export class ReportRepository {
  constructor(private dataSource: DatabaseService) {}

  async getPesawat() {
    try {
      const result = await selectPesawat.run(undefined, this.dataSource);
      return result;
    } catch (error) {
      throw error;
    }
  }
}
```

## Prepare Dto

create a file named `report/report.dto.ts` inside folder `report` and add this following script :

```typescript
export interface ReportDto {
  reportDate: Date;
  attachmentNumber: string;
  headerTitle: string;
  reportTitle: string;
  titleSigner: string;
  isAliases: boolean;
  positionHonorable: string[];
  ccLetter: string[];
  nrpSigner: string;
  nrpSignerAlias: string;
  personelName: string;
  personelNameAlias: string;
  baseReport: string;
}
```

## Prepare Service

add this following script into `report/report.service.ts` :

```typescript
import { Injectable } from '@nestjs/common';
import { ReportRepository } from './report.repository';
import { ReportDto } from './report.dto';
import { createPdf } from '../utils/generate.pdf';
import * as path from 'path';

@Injectable()
export class ReportService {
  constructor(private reportRepository: ReportRepository) {}

  async getPesawat(reportDto: ReportDto) {
    try {
      let data = {
        items: await this.reportRepository.getPesawat(),
        info: reportDto,
        position: reportDto.position.join(' '),
        signer:
          reportDto.isAliases === true
            ? reportDto.personelName
            : reportDto.personelNameAlias,
        nrpSigner:
          reportDto.isAliases === true
            ? reportDto.nrpSigner
            : reportDto.nrpSignerAlias,
        attachmentNumber: reportDto.attachmentNumber
          ? 'Nomor : ' + reportDto.attachmentNumber
          : '',
        honorable:
          reportDto.positionHonorable.length === 1
            ? reportDto.positionHonorable.map((position) => ({
                index: '',
                name: position,
              }))
            : reportDto.positionHonorable.map((position, index) => ({
                index: index + 1 + '.',
                name: position,
              })),
        ccLetter:
          reportDto.ccLetter.length === 1
            ? reportDto.ccLetter.map((position) => ({
                index: '',
                name: position,
              }))
            : reportDto.ccLetter.map((position, index) => ({
                index: index + 1 + '.',
                name: position,
              })),
      };

      const file = path.join(process.cwd(), 'templates', 'laporan.hbs');
      return await createPdf(file, undefined, data);
    } catch (error) {
      throw error;
    }
  }
}
```

## Prepare Template Hbs

create a file named `templates/laporan.hbs` inside folder `templates` and add this following script :

```hbs
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style type="text/css">

    table {
      border-collapse: collapse;
      width: 100%;
    }

    .underline-text {
      font-size: 18px;
      border-bottom: 2px solid #000; /* You can set the color you prefer */
      display: inline-block;
    }

    .header {
      font-weight: bold;
      text-align: left;
      position: relative; /* Add this for positioning */
    }

    .border-line {
      position: absolute;
      bottom: 0;
      left: 0;
      content: "";
      width: 100%;
      border-bottom: 1px solid;
      padding-bottom: 5px;
      margin-bottom: 10px;
    }

    .date {
      font-weight: bold;
    }

    .center {
      text-align: center;
    }


    tbody {
      border-collapse: collapse; /* Collapse borders within the tbody */
    }


    tbody td {
      border: none; /* Removes any cell-specific borders */
    }

    .border-line {
      position: absolute;
      bottom: 0;
      left: 0;
      content: "";
      width: 100%;
      border-bottom: 1px solid;
      padding-bottom: 5px;
      margin-bottom: 10px;
    }

    .additional-content {
      margin-top: 10px;
      display: flex;
      justify-content: space-between;
    }

    .left-content {
      text-align: left;
    }

    .right-content {
      text-align: right;
    }

  .no-border {
    border-left: 1px solid black;
    border-right: 1px solid black;
  }

  .no-right-border {
    border-right: none;
  }

  html {
      font-family: Arial, sans-serif;
      font-size: 10px;
    }

  .container {
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      width: fit-content;
  }
  .section {
      text-align: center;
      margin-bottom: 10px;
  }
  .section-honor {
      text-align: left;
      margin-bottom: 10px;
  }
  .header1 {
      font-size: 11px;
      font-weight: bold;
  }
  .line {
      width: 100%;
     border-bottom: 2px solid #000;
      margin-top: 0.1px;
  }

  th, td {
      border: 1px solid;
      padding: 8px;
    }



    tfoot {
      display: table-footer-group;
    }

    .page-break-before {
      page-break-before: always;
    }


  </style>
</head>
<script>
    function onChange(element) {
        const pageNumberElement = element.getElementsByClassName('pageNumber')[0];
        if (pageNumberElement.textContent.trim() === '1') {
            element.style.display = 'none'
        }
    }
</script>
<body>

  <div id="app">
      <div style="width: 100%; text-align: center; font-size: 10px; margin-bottom:20px; margin-left:-17px">
        <strong>RAHASIA</strong>
      </div>

      <div id="header" style="width: 100%; text-align: center; font-size: 10px; margin-bottom:20px; margin-left:-17px" onchange="onChange(this)">
        <strong>RAHASIA</strong>
        <br>
        <span class="pageNumber">
      </div>

    <div class="container" >
        <div class="section" >
            <div class="header1">MARKAS BESAR TNI ANGKATAN UDARA</div>
            <div class="header1">PUSAT KOMANDO DAN PENGENDALIAN</div>
            <div class="line"></div>
        </div>
        <br>
        <p style="font-size: 12px !important;"><strong>{{attachmentNumber}}</strong></p>
    </div>


    <div class="center" style="margin: 0; padding: 0;">

      <h3 style="margin: 0;"><strong>{{info.reportTitle}} {{info.headerTitle}}</strong></h3>
      <p style="font-size: 10px !important; margin: 0;">{{info.reportDate}}</p>
      <br>
      <br>
    </div>


    <table border="1">
      <thead>
        <tr>
          <th style="width: 1%; text-align: center;">No.</th>
          <th style="width: 10%; text-align: center;">REG. NO</th>
          <th style="width: 20%; text-align: center;">LANUD</th>
          <th style="width: 15%; text-align: center;">KOOPSUD</th>
        </tr>
        <tr style="height: 1px !important; padding: 1px !important;">
          <th style="width: 1%; text-align: center; padding: 1px !important;">1</th>
          <th style="width: 20%; text-align: center; padding: 1px !important;">2</th>
          <th style="width: 15%; text-align: center; padding: 1px !important;">3</th>
          <th style="width: 10%; text-align: center; padding: 1px !important;">4</th>
        </tr>
      </thead>
      <tbody>
        {{#each items}}
          <tr>
            <td style="text-align: center;" class="no-border">{{no}}</td>
            <td style="text-align: left;" style="width: 1%;" class="no-border">{{regnumber}}</td>
            <td style="text-align: left;" class="no-border">{{lanud}}</td>
            <td style="text-align: center;" class="no-border">{{koopsud}}</td>
          </tr>
        {{/each}}
      </tbody>
      <tfoot style="height: 0px; padding: 0px;">
        <tr style="height: 0px; padding: 0px;">
        </tr>
      </tfoot>
    </table>

    <div class="additional-content">
      <div class="left-content" style="margin-top: 90px !important;">
        <p><strong>Kepada Yth :</strong></p>

        <div class="container">
          <div class="section-honor">
            {{#each honorable}}
            <p style="margin-bottom: 0 !important;">{{this.index}} {{this.name}}</p>
            {{/each}}
          </div>
        </div>

        <br>

        {{#if ccLetter}}
        <p><strong>Tembusan :</strong></p>
        <div class="container">
          <div class="section-honor">
            {{#each ccLetter}}
            <p style="margin-bottom: 0 !important;">{{this.index}} {{this.name}}</p>
            {{/each}}
            <div class="line"></div>
          </div>
        </div>
        {{/if}}
      </div>

      <div class="right-content">
        {{#if isalias}}
        <br>
        <p style="text-align:center;">a.n. {{join position " "}},</p>
        <br>
        <br>
        <br>
          {{!-- <p style="text-align:center;">{{signer}}<br/>{{signerPositionKorps}}</p> --}}
        <p style="text-align:center;">{{info.nrpSigner}}</p>
        <p style="text-align:center;">{{info.nrpSignerAlias}}</p>
        {{else}}
        <br>
        <p style="text-align:center;">{{{position}}},</p>
        <br>
        <br>
        <br>
        {{!-- <p style="text-align:center;">{{signer}}<br/>{{signerPositionKorps}}</p> --}}
        <p style="text-align:center;">{{info.nrpSigner}}</p>
        <p style="text-align:center;">{{info.personelName}}</p>
        {{/if}}
      </div>

    </div>

  </div>

</body>
</html>

```

## Prepare Route

add this following script into `report/report.controller.ts`

```typescript
import { Body, Controller, HttpStatus, Post, Req, Res } from '@nestjs/common';
import { ReportService } from './report.service';
import { ReportDto } from './report.dto';

@Controller('report')
export class ReportController {
  constructor(private readonly reportService: ReportService) {}

  @Post('/print')
  async print(@Res() res, @Body() body: ReportDto) {
    try {
      let buffer;
      let filename;

      buffer = await this.reportService.getPesawat(body);
      filename = `${body.reportTitle.replaceAll(' ', '') + `.pdf`}`;

      res.set({
        'Content-Type': 'application/pdf',
        'Content-Disposition': `attachment; filename=${filename}`,
        'Content-Length': buffer.length.toString(),
        'Cache-Control': 'no-cache, no-store, must-revalidate',
        Pragma: 'no-cache',
        Expires: '0',
      });

      res.end(buffer);
    } catch (error) {
      res
        .status(HttpStatus.INTERNAL_SERVER_ERROR)
        .json({ message: error.message });
    }
  }
}
```

## Test the endpoint

a body to test the endpoint :

```json
{
  "reportDate": "2024-07-14",
  "headerTitle": "",
  "reportTitle": "Laporan Harian",
  "isWeekday": 1,
  "isAliases": true,
  "attachmentNumber": "123",
  "positionHonorable": [],
  "ccLetter": [],
  "nrpSigner": "",
  "nrpSignerAlias": "",
  "personelName": "",
  "personelNameAlias": "",
  "position": []
}
```

# Setup Master

- run this following script `nest g resource pesawat --no-spec`

## Prepare Query

- create a file named `pesawat/pesawat.queries.ts` inside folder `pesawat` and add this following script :

```typescript
import { sql } from '@pgtyped/runtime';
import {
  ISelectPesawatQuery,
  IInsertPesawatQuery,
  IUpdatePesawatByIdQuery,
  IDeletePesawatByIdQuery,
  ISelectPesawatByIdQuery,
} from './pesawat.queries.types';

export const selectPesawat = sql<ISelectPesawatQuery>`
  SELECT
    id,
    regNumber,
    lanud,
    koopsud
  FROM
    pesawat
  `;

export const selectPesawatById = sql<ISelectPesawatByIdQuery>`
  SELECT
    id,
    regNumber,
    lanud,
    koopsud
  FROM
    pesawat
  WHERE
    id = $id
  `;

export const insertPesawat = sql<IInsertPesawatQuery>`
    INSERT INTO
        pesawat (regNumber, lanud, koopsud)
    VALUES
        $pesawat(regNumber, lanud, koopsud)
`;

export const updatePesawatById = sql<IUpdatePesawatByIdQuery>`
    UPDATE
        pesawat
    SET
        lanud = $lanud,
        koopsud = $koopsud,
        regNumber = $regNumber
    WHERE
        id = $id
`;

export const deletePesawatById = sql<IDeletePesawatByIdQuery>`
    DELETE FROM
        pesawat
    WHERE
        id = $id
`;
```

## Prepare Repository

- create a file named `pesawat/pesawat.repository.ts` inside folder `pesawat` and add this following script :

```typescript
import { Injectable } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import {
  selectPesawat,
  deletePesawatById,
  insertPesawat,
  updatePesawatById,
  selectPesawatById,
} from './pesawat.queries';

@Injectable()
export class PesawatRepository {
  constructor(private dataSource: DatabaseService) {}

  async getPesawatAll() {
    try {
      const result = await selectPesawat.run(undefined, this.dataSource);
      return result;
    } catch (error) {
      throw error;
    }
  }

  async insertPesawat(regNumber: string, lanud: string, koopsud: string) {
    try {
      const result = await insertPesawat.run(
        { pesawat: { regNumber, lanud, koopsud } },
        this.dataSource,
      );
      return result;
    } catch (error) {
      throw error;
    }
  }

  async updatePesawatById(
    id: number,
    regNumber: string,
    lanud: string,
    koopsud: string,
  ) {
    try {
      const result = await updatePesawatById.run(
        { id, regNumber, lanud, koopsud },
        this.dataSource,
      );
      return result;
    } catch (error) {
      throw error;
    }
  }

  async deletePesawatById(id: number) {
    try {
      const result = await deletePesawatById.run({ id }, this.dataSource);
      return result;
    } catch (error) {
      throw error;
    }
  }

  async selectPesawatById(id: number) {
    try {
      const result = await selectPesawatById.run({ id }, this.dataSource);
      return result;
    } catch (error) {
      throw error;
    }
  }
}
```

## Prepare Dto

- create a file named `pesawat/pesawat.dto.ts` inside folder `pesawat` and add this following script :

```typescript
export interface PesawatDto {
  id: number;
  regNumber: string;
  lanud: string;
  koopsud: string;
}
```

## Prepare Service

- add this following script into `pesawat/pesawat.service.ts` :

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { PesawatRepository } from './pesawat.repository';
import { PesawatDto } from './pesawat.dto';

@Injectable()
export class PesawatService {
  constructor(private pesawatRepository: PesawatRepository) {}

  async getPesawatAll() {
    try {
      const data = await this.pesawatRepository.getPesawatAll();
      return {
        data,
      };
    } catch (error) {
      throw error;
    }
  }

  async insertPesawat(req: PesawatDto) {
    try {
      let pesawat = await this.pesawatRepository.insertPesawat(
        req.regNumber,
        req.lanud,
        req.koopsud,
      );
      return pesawat;
    } catch (error) {
      throw error;
    }
  }

  async updatePesawatById(id: number, req: PesawatDto) {
    try {
      let data = await this.pesawatRepository.selectPesawatById(id);

      if (data.length === 0) {
        throw new Error('Pesawat not found.');
      }

      let pesawat = await this.pesawatRepository.updatePesawatById(
        id,
        req.regNumber,
        req.lanud,
        req.koopsud,
      );
      return pesawat;
    } catch (error) {
      throw error;
    }
  }

  async deletePesawatById(id: number) {
    try {
      let data = await this.pesawatRepository.selectPesawatById(id);

      if (data.length === 0) {
        throw new NotFoundException('Data not found.');
      }

      let pesawat = await this.pesawatRepository.deletePesawatById(id);
      return pesawat;
    } catch (error) {
      throw error;
    }
  }
}
```

## Prepare Route

- add this following script into `pesawat/pesawat.controller.ts`

```typescript
import {
  Body,
  Controller,
  Delete,
  Get,
  Param,
  Post,
  Put,
  Query,
  SetMetadata,
  UseFilters,
  UseGuards,
} from '@nestjs/common';
import { PesawatService } from './pesawat.service';
import { PesawatDto } from './pesawat.dto';

@Controller('pesawat')
export class PesawatController {
  constructor(private readonly pesawatService: PesawatService) {}

  @Post('insert')
  async createPesawat(@Body() pesawatDto: PesawatDto) {
    await this.pesawatService.insertPesawat(pesawatDto);
    return {
      success: true,
      statusCode: 201,
      message: 'Pesawat created successfully',
    };
  }

  @Put('update/:id')
  async updatePesawat(@Param('id') id: number, @Body() pesawatDto: PesawatDto) {
    await this.pesawatService.updatePesawatById(id, pesawatDto);
    return {
      success: true,
      statusCode: 200,
      message: 'Pesawat updated successfully',
    };
  }

  @Delete('delete/:id')
  async deletePesawat(@Param('id') id: number) {
    await this.pesawatService.deletePesawatById(id);
    return {
      success: true,
      statusCode: 200,
      message: 'Pesawat deleted successfully',
    };
  }

  @Get('get-all')
  async getAllPesawat() {
    let data = await this.pesawatService.getPesawatAll();
    return { success: true, statusCode: 200, data };
  }
}
```
