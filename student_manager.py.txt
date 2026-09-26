from __future__ import annotations

import argparse
import csv
import json
import re
import sqlite3
import sys
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path
from typing import Any, Iterable, Optional


# ============================================================
# CONSTANTS
# ============================================================

DEFAULT_DB = "students.db"

EMAIL_PATTERN = re.compile(
    r"^[A-Za-z0-9.!#$%&'*+/=?^_`{|}~-]+@"
    r"[A-Za-z0-9](?:[A-Za-z0-9-]{0,61}[A-Za-z0-9])?"
    r"(?:\.[A-Za-z0-9](?:[A-Za-z0-9-]{0,61}[A-Za-z0-9])?)+$"
)

ROLL_PATTERN = re.compile(
    r"^[A-Za-z0-9][A-Za-z0-9._/-]{0,39}$"
)

# Changed {1,99} to {0,99} so one-character names are allowed.
NAME_PATTERN = re.compile(
    r"^[A-Za-z][A-Za-z .'-]{0,99}$"
)

GRADE_PATTERN = re.compile(
    r"^[A-Za-z0-9][A-Za-z0-9 +._-]{0,19}$"
)

TABLE_COLUMNS = (
    "id",
    "roll_number",
    "name",
    "email",
    "age",
    "grade",
    "gpa",
    "created_at",
    "updated_at",
)


# ============================================================
# CUSTOM EXCEPTIONS
# ============================================================

class StudentManagerError(Exception):
    """Base exception for the application."""


class ValidationError(StudentManagerError):
    """Raised when user-provided data fails validation."""


class DuplicateStudentError(StudentManagerError):
    """Raised when a unique student field already exists."""


class StudentNotFoundError(StudentManagerError):
    """Raised when a requested student cannot be found."""


class DatabaseError(StudentManagerError):
    """Raised when a database operation fails."""


# ============================================================
# DATA MODEL
# ============================================================

@dataclass(frozen=True)
class Student:
    """Immutable representation of a student."""

    id: Optional[int]
    roll_number: str
    name: str
    email: str
    age: int
    grade: str
    gpa: float
    created_at: str = ""
    updated_at: str = ""

    @classmethod
    def from_row(cls, row: sqlite3.Row) -> "Student":
        """Create Student object from SQLite row."""
        return cls(
            id=row["id"],
            roll_number=row["roll_number"],
            name=row["name"],
            email=row["email"],
            age=row["age"],
            grade=row["grade"],
            gpa=row["gpa"],
            created_at=row["created_at"],
            updated_at=row["updated_at"],
        )


# ============================================================
# VALIDATION
# ============================================================

class Validator:
    """Validation and normalization rules."""

    @staticmethod
    def roll_number(value: str) -> str:
        value = value.strip().upper()

        if not value:
            raise ValidationError(
                "Roll/registration number cannot be empty."
            )

        if not ROLL_PATTERN.fullmatch(value):
            raise ValidationError(
                "Roll number may contain letters, numbers, '.', '_', '/', "
                "and '-' only, with a maximum of 40 characters."
            )

        return value

    @staticmethod
    def name(value: str) -> str:
        value = " ".join(value.strip().split())

        if not value:
            raise ValidationError("Name cannot be empty.")

        if not NAME_PATTERN.fullmatch(value):
            raise ValidationError(
                "Name must contain letters and may include spaces, "
                "apostrophes, periods, or hyphens."
            )

        return value.title()

    @staticmethod
    def email(value: str) -> str:
        value = value.strip().lower()

        if not value:
            raise ValidationError("Email cannot be empty.")

        if len(value) > 254 or not EMAIL_PATTERN.fullmatch(value):
            raise ValidationError("Please enter a valid email address.")

        return value

    @staticmethod
    def age(value: Any) -> int:
        try:
            age = int(str(value).strip())
        except (TypeError, ValueError) as exc:
            raise ValidationError(
                "Age must be a whole number."
            ) from exc

        if not 1 <= age <= 120:
            raise ValidationError(
                "Age must be between 1 and 120."
            )

        return age

    @staticmethod
    def grade(value: str) -> str:
        value = " ".join(value.strip().split())

        if not value:
            raise ValidationError(
                "Grade/Class cannot be empty."
            )

        if not GRADE_PATTERN.fullmatch(value):
            raise ValidationError(
                "Grade/Class contains unsupported characters."
            )

        return value.upper()

    @staticmethod
    def gpa(value: Any) -> float:
        try:
            number = float(str(value).strip())
        except (TypeError, ValueError) as exc:
            raise ValidationError(
                "GPA/Marks must be a numeric value."
            ) from exc

        if not 0 <= number <= 100:
            raise ValidationError(
                "GPA/Marks must be between 0 and 100."
            )

        return round(number, 2)

    @staticmethod
    def student_id(value: Any) -> int:
        """Validate a database student ID."""
        try:
            student_id = int(str(value).strip())
        except (TypeError, ValueError) as exc:
            raise ValidationError(
                "Student ID must be a whole number."
            ) from exc

        if student_id <= 0:
            raise ValidationError(
                "Student ID must be greater than zero."
            )

        return student_id


# ============================================================
# DATABASE
# ============================================================

class DatabaseManager:
    """Manages SQLite database connection and initialization."""

    def __init__(self, db_path: str = DEFAULT_DB) -> None:
        self.db_path = Path(db_path).expanduser()
        self._initialize_database()

    def connect(self) -> sqlite3.Connection:
        try:
            connection = sqlite3.connect(
                self.db_path,
                timeout=10,
            )

            connection.row_factory = sqlite3.Row
            connection.execute(
                "PRAGMA foreign_keys = ON"
            )

            return connection

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to connect to database: {exc}"
            ) from exc

    def _initialize_database(self) -> None:
        try:
            if self.db_path.parent != Path("."):
                self.db_path.parent.mkdir(
                    parents=True,
                    exist_ok=True,
                )

            with self.connect() as connection:

                connection.execute(
                    """
                    CREATE TABLE IF NOT EXISTS students (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        roll_number TEXT NOT NULL UNIQUE,
                        name TEXT NOT NULL,
                        email TEXT NOT NULL UNIQUE,
                        age INTEGER NOT NULL
                            CHECK(age BETWEEN 1 AND 120),
                        grade TEXT NOT NULL,
                        gpa REAL NOT NULL
                            CHECK(gpa BETWEEN 0 AND 100),
                        created_at TEXT NOT NULL,
                        updated_at TEXT NOT NULL
                    )
                    """
                )

                # Additional indexes for faster searching.
                connection.execute(
                    """
                    CREATE INDEX IF NOT EXISTS idx_students_name
                    ON students(name)
                    """
                )

                connection.execute(
                    """
                    CREATE INDEX IF NOT EXISTS idx_students_grade
                    ON students(grade)
                    """
                )

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to initialize database: {exc}"
            ) from exc


# ============================================================
# REPOSITORY
# ============================================================

class StudentRepository:
    """Database operations for student records."""

    def __init__(self, database: DatabaseManager) -> None:
        self.database = database

    @staticmethod
    def _now() -> str:
        return datetime.now().astimezone().isoformat(
            timespec="seconds"
        )

    @staticmethod
    def _translate_integrity_error(
        exc: sqlite3.IntegrityError,
    ) -> None:

        message = str(exc).lower()

        if "roll_number" in message:
            raise DuplicateStudentError(
                "That roll/registration number already exists."
            ) from exc

        if "email" in message:
            raise DuplicateStudentError(
                "That email address is already registered."
            ) from exc

        raise DatabaseError(
            f"Database constraint error: {exc}"
        ) from exc

    def create(self, student: Student) -> Student:
        now = self._now()

        try:
            with self.database.connect() as connection:

                cursor = connection.execute(
                    """
                    INSERT INTO students (
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        created_at,
                        updated_at
                    )
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                    """,
                    (
                        student.roll_number,
                        student.name,
                        student.email,
                        student.age,
                        student.grade,
                        student.gpa,
                        now,
                        now,
                    ),
                )

                student_id = cursor.lastrowid

            return Student(
                id=student_id,
                roll_number=student.roll_number,
                name=student.name,
                email=student.email,
                age=student.age,
                grade=student.grade,
                gpa=student.gpa,
                created_at=now,
                updated_at=now,
            )

        except sqlite3.IntegrityError as exc:
            self._translate_integrity_error(exc)

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to create student: {exc}"
            ) from exc

        raise DatabaseError(
            "Unable to create student."
        )

    def get_all(self) -> list[Student]:
        try:
            with self.database.connect() as connection:
                rows = connection.execute(
                    """
                    SELECT
                        id,
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        created_at,
                        updated_at
                    FROM students
                    ORDER BY id ASC
                    """
                ).fetchall()

            return [
                Student.from_row(row)
                for row in rows
            ]

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to read students: {exc}"
            ) from exc

    def get_by_id(
        self,
        student_id: int,
    ) -> Optional[Student]:

        try:
            with self.database.connect() as connection:
                row = connection.execute(
                    """
                    SELECT
                        id,
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        created_at,
                        updated_at
                    FROM students
                    WHERE id = ?
                    """,
                    (student_id,),
                ).fetchone()

            return (
                Student.from_row(row)
                if row
                else None
            )

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to find student: {exc}"
            ) from exc

    def get_by_roll(
        self,
        roll_number: str,
    ) -> Optional[Student]:

        try:
            with self.database.connect() as connection:
                row = connection.execute(
                    """
                    SELECT
                        id,
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        created_at,
                        updated_at
                    FROM students
                    WHERE roll_number = ?
                    """,
                    (roll_number,),
                ).fetchone()

            return (
                Student.from_row(row)
                if row
                else None
            )

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to find student: {exc}"
            ) from exc

    def search(self, query: str) -> list[Student]:
        query = query.strip()
        like_query = f"%{query}%"

        try:
            with self.database.connect() as connection:
                rows = connection.execute(
                    """
                    SELECT
                        id,
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        created_at,
                        updated_at
                    FROM students
                    WHERE name LIKE ?
                       OR email LIKE ?
                       OR roll_number LIKE ?
                       OR grade LIKE ?
                    ORDER BY name COLLATE NOCASE ASC
                    """,
                    (
                        like_query,
                        like_query,
                        like_query,
                        like_query,
                    ),
                ).fetchall()

            return [
                Student.from_row(row)
                for row in rows
            ]

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to search students: {exc}"
            ) from exc

    def update(
        self,
        student_id: int,
        roll_number: str,
        name: str,
        email: str,
        age: int,
        grade: str,
        gpa: float,
    ) -> Student:

        now = self._now()

        try:
            with self.database.connect() as connection:

                cursor = connection.execute(
                    """
                    UPDATE students
                    SET
                        roll_number = ?,
                        name = ?,
                        email = ?,
                        age = ?,
                        grade = ?,
                        gpa = ?,
                        updated_at = ?
                    WHERE id = ?
                    """,
                    (
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        now,
                        student_id,
                    ),
                )

                if cursor.rowcount == 0:
                    raise StudentNotFoundError(
                        "Student record was not found."
                    )

                row = connection.execute(
                    """
                    SELECT
                        id,
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        created_at,
                        updated_at
                    FROM students
                    WHERE id = ?
                    """,
                    (student_id,),
                ).fetchone()

            if row is None:
                raise StudentNotFoundError(
                    "Student record was not found."
                )

            return Student.from_row(row)

        except sqlite3.IntegrityError as exc:
            self._translate_integrity_error(exc)

        except StudentNotFoundError:
            raise

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to update student: {exc}"
            ) from exc

        raise DatabaseError(
            "Unable to update student."
        )

    def delete(self, student_id: int) -> None:

        try:
            with self.database.connect() as connection:

                cursor = connection.execute(
                    "DELETE FROM students WHERE id = ?",
                    (student_id,),
                )

                if cursor.rowcount == 0:
                    raise StudentNotFoundError(
                        "Student record was not found."
                    )

        except StudentNotFoundError:
            raise

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to delete student: {exc}"
            ) from exc

    def statistics(self) -> dict[str, Any]:

        try:
            with self.database.connect() as connection:

                total = connection.execute(
                    "SELECT COUNT(*) FROM students"
                ).fetchone()[0]

                average_gpa = connection.execute(
                    "SELECT AVG(gpa) FROM students"
                ).fetchone()[0]

                top = connection.execute(
                    """
                    SELECT
                        id,
                        roll_number,
                        name,
                        email,
                        age,
                        grade,
                        gpa,
                        created_at,
                        updated_at
                    FROM students
                    ORDER BY gpa DESC,
                             name COLLATE NOCASE ASC
                    LIMIT 1
                    """
                ).fetchone()

                distribution_rows = connection.execute(
                    """
                    SELECT
                        grade,
                        COUNT(*) AS count
                    FROM students
                    GROUP BY grade
                    ORDER BY count DESC, grade ASC
                    """
                ).fetchall()

            return {
                "total": total,
                "average_gpa": round(
                    average_gpa or 0.0,
                    2,
                ),
                "top_student": (
                    Student.from_row(top)
                    if top
                    else None
                ),
                "grade_distribution": {
                    row["grade"]: row["count"]
                    for row in distribution_rows
                },
            }

        except sqlite3.Error as exc:
            raise DatabaseError(
                f"Unable to calculate statistics: {exc}"
            ) from exc


# ============================================================
# EXPORT SERVICE
# ============================================================

class ExportService:

    @staticmethod
    def _records(
        students: Iterable[Student],
    ) -> list[dict[str, Any]]:

        return [
            {
                "id": student.id,
                "roll_number": student.roll_number,
                "name": student.name,
                "email": student.email,
                "age": student.age,
                "grade": student.grade,
                "gpa": student.gpa,
                "created_at": student.created_at,
                "updated_at": student.updated_at,
            }
            for student in students
        ]

    @classmethod
    def to_json(
        cls,
        students: Iterable[Student],
        path: str,
    ) -> Path:

        output = Path(path).expanduser()

        try:
            if output.parent != Path("."):
                output.parent.mkdir(
                    parents=True,
                    exist_ok=True,
                )

            with output.open(
                "w",
                encoding="utf-8",
            ) as file:

                json.dump(
                    cls._records(students),
                    file,
                    indent=4,
                    ensure_ascii=False,
                )

            return output

        except OSError as exc:
            raise StudentManagerError(
                f"Unable to export JSON: {exc}"
            ) from exc

    @classmethod
    def to_csv(
        cls,
        students: Iterable[Student],
        path: str,
    ) -> Path:

        output = Path(path).expanduser()
        records = cls._records(students)

        try:
            if output.parent != Path("."):
                output.parent.mkdir(
                    parents=True,
                    exist_ok=True,
                )

            with output.open(
                "w",
                newline="",
                encoding="utf-8",
            ) as file:

                writer = csv.DictWriter(
                    file,
                    fieldnames=list(TABLE_COLUMNS),
                )

                writer.writeheader()
                writer.writerows(records)

            return output

        except OSError as exc:
            raise StudentManagerError(
                f"Unable to export CSV: {exc}"
            ) from exc


# ============================================================
# TERMINAL UI
# ============================================================

class TerminalUI:

    WIDTH = 86

    @staticmethod
    def title(text: str) -> None:
        print("\n" + "=" * TerminalUI.WIDTH)
        print(text.center(TerminalUI.WIDTH))
        print("=" * TerminalUI.WIDTH)

    @staticmethod
    def section(text: str) -> None:
        print("\n" + "-" * TerminalUI.WIDTH)
        print(f"  {text}")
        print("-" * TerminalUI.WIDTH)

    @staticmethod
    def success(message: str) -> None:
        print(f"\n[OK] {message}")

    @staticmethod
    def error(message: str) -> None:
        print(f"\n[ERROR] {message}")

    @staticmethod
    def warning(message: str) -> None:
        print(f"\n[WARNING] {message}")

    @staticmethod
    def info(message: str) -> None:
        print(f"\n[INFO] {message}")

    @staticmethod
    def prompt(message: str) -> str:
        try:
            return input(message)
        except EOFError:
            print()
            raise KeyboardInterrupt

    @staticmethod
    def pause() -> None:
        try:
            input("\nPress Enter to continue...")
        except EOFError:
            pass

    @staticmethod
    def display_student(
        student: Student,
    ) -> None:

        print(
            f"""
  ID          : {student.id}
  Roll Number : {student.roll_number}
  Name        : {student.name}
  Email       : {student.email}
  Age         : {student.age}
  Grade/Class : {student.grade}
  GPA/Marks   : {student.gpa:.2f}
  Created At  : {student.created_at}
  Updated At  : {student.updated_at}
"""
        )

    @staticmethod
    def display_table(
        students: list[Student],
    ) -> None:

        if not students:
            TerminalUI.info(
                "No student records found."
            )
            return

        headers = [
            "ID",
            "Roll Number",
            "Name",
            "Email",
            "Age",
            "Grade",
            "GPA",
        ]

        rows = [
            [
                str(s.id),
                s.roll_number,
                s.name,
                s.email,
                str(s.age),
                s.grade,
                f"{s.gpa:.2f}",
            ]
            for s in students
        ]

        widths = [
            len(header)
            for header in headers
        ]

        for row in rows:
            for index, value in enumerate(row):
                widths[index] = min(
                    max(
                        widths[index],
                        len(value),
                    ),
                    28,
                )

        def format_row(
            row: list[str],
        ) -> str:

            cells = []

            for index, value in enumerate(row):
                width = widths[index]

                if len(value) > width:
                    value = (
                        value[: width - 1]
                        + "…"
                    )

                cells.append(
                    value.ljust(width)
                )

            return " | ".join(cells)

        separator = "-+-".join(
            "-" * width
            for width in widths
        )

        print()
        print(format_row(headers))
        print(separator)

        for row in rows:
            print(format_row(row))

        print(separator)
        print(
            f"Total records displayed: "
            f"{len(students)}"
        )


# ============================================================
# APPLICATION
# ============================================================

class StudentApplication:

    def __init__(
        self,
        repository: StudentRepository,
    ) -> None:

        self.repository = repository

    def _validated_input(
        self,
        label: str,
        validator,
        current: Optional[str] = None,
    ) -> Any:

        while True:

            suffix = (
                f" [{current}]"
                if current is not None
                else ""
            )

            value = TerminalUI.prompt(
                f"{label}{suffix}: "
            ).strip()

            if (
                current is not None
                and value == ""
            ):
                value = current

            try:
                return validator(value)

            except ValidationError as exc:
                TerminalUI.error(str(exc))

    def _input_new_student(self) -> Student:

        TerminalUI.section(
            "ADD NEW STUDENT"
        )

        roll_number = self._validated_input(
            "Roll/Registration Number",
            Validator.roll_number,
        )

        name = self._validated_input(
            "Name",
            Validator.name,
        )

        email = self._validated_input(
            "Email",
            Validator.email,
        )

        age = self._validated_input(
            "Age",
            Validator.age,
        )

        grade = self._validated_input(
            "Grade/Class",
            Validator.grade,
        )

        gpa = self._validated_input(
            "GPA/Marks (0-100)",
            Validator.gpa,
        )

        return Student(
            id=None,
            roll_number=roll_number,
            name=name,
            email=email,
            age=age,
            grade=grade,
            gpa=gpa,
        )

    # --------------------------------------------------------
    # ADD
    # --------------------------------------------------------

    def add_student(self) -> None:

        try:
            student = self._input_new_student()

            created = self.repository.create(
                student
            )

            TerminalUI.success(
                f"Student '{created.name}' "
                f"added successfully with ID "
                f"{created.id}."
            )

        except DuplicateStudentError as exc:
            TerminalUI.error(str(exc))

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

        except KeyboardInterrupt:
            TerminalUI.warning(
                "Operation cancelled."
            )

    # --------------------------------------------------------
    # VIEW ALL
    # --------------------------------------------------------

    def view_all(self) -> None:

        try:
            students = self.repository.get_all()

            TerminalUI.section(
                "ALL STUDENT RECORDS"
            )

            TerminalUI.display_table(
                students
            )

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

    # --------------------------------------------------------
    # VIEW SINGLE
    # --------------------------------------------------------

    def view_single(self) -> None:

        TerminalUI.section(
            "VIEW STUDENT"
        )

        choice = TerminalUI.prompt(
            "Find by [1] ID  [2] Roll Number: "
        ).strip()

        try:

            if choice == "1":

                student_id = Validator.student_id(
                    TerminalUI.prompt(
                        "Student ID: "
                    )
                )

                student = (
                    self.repository.get_by_id(
                        student_id
                    )
                )

            elif choice == "2":

                roll = Validator.roll_number(
                    TerminalUI.prompt(
                        "Roll Number: "
                    )
                )

                student = (
                    self.repository.get_by_roll(
                        roll
                    )
                )

            else:
                TerminalUI.error(
                    "Invalid option."
                )
                return

            if student is None:
                raise StudentNotFoundError(
                    "Student record not found."
                )

            TerminalUI.display_student(
                student
            )

        except ValidationError as exc:
            TerminalUI.error(str(exc))

        except StudentNotFoundError as exc:
            TerminalUI.warning(str(exc))

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

    # --------------------------------------------------------
    # SEARCH
    # --------------------------------------------------------

    def search_students(self) -> None:

        TerminalUI.section(
            "SEARCH STUDENTS"
        )

        query = TerminalUI.prompt(
            "Enter name, email, roll number or grade: "
        ).strip()

        if not query:
            TerminalUI.error(
                "Search text cannot be empty."
            )
            return

        try:
            students = self.repository.search(
                query
            )

            TerminalUI.display_table(
                students
            )

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

    # --------------------------------------------------------
    # UPDATE
    # --------------------------------------------------------

    def update_student(self) -> None:

        TerminalUI.section(
            "UPDATE STUDENT"
        )

        lookup = TerminalUI.prompt(
            "Enter Student ID or Roll Number: "
        ).strip()

        try:

            if lookup.isdigit():

                student_id = (
                    Validator.student_id(
                        lookup
                    )
                )

                student = (
                    self.repository.get_by_id(
                        student_id
                    )
                )

            else:

                roll = Validator.roll_number(
                    lookup
                )

                student = (
                    self.repository.get_by_roll(
                        roll
                    )
                )

            if student is None:
                raise StudentNotFoundError(
                    "Student record not found."
                )

            TerminalUI.info(
                "Press Enter to keep the current value."
            )

            roll_number = self._validated_input(
                "Roll/Registration Number",
                Validator.roll_number,
                student.roll_number,
            )

            name = self._validated_input(
                "Name",
                Validator.name,
                student.name,
            )

            email = self._validated_input(
                "Email",
                Validator.email,
                student.email,
            )

            age = self._validated_input(
                "Age",
                Validator.age,
                str(student.age),
            )

            grade = self._validated_input(
                "Grade/Class",
                Validator.grade,
                student.grade,
            )

            gpa = self._validated_input(
                "GPA/Marks (0-100)",
                Validator.gpa,
                str(student.gpa),
            )

            updated = self.repository.update(
                student.id,
                roll_number,
                name,
                email,
                age,
                grade,
                gpa,
            )

            TerminalUI.success(
                f"Student '{updated.name}' "
                f"updated successfully."
            )

        except ValidationError as exc:
            TerminalUI.error(str(exc))

        except StudentNotFoundError as exc:
            TerminalUI.warning(str(exc))

        except DuplicateStudentError as exc:
            TerminalUI.error(str(exc))

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

        except KeyboardInterrupt:
            TerminalUI.warning(
                "Update cancelled."
            )

    # --------------------------------------------------------
    # DELETE
    # --------------------------------------------------------

    def delete_student(self) -> None:

        TerminalUI.section(
            "DELETE STUDENT"
        )

        lookup = TerminalUI.prompt(
            "Enter Student ID or Roll Number: "
        ).strip()

        try:

            if lookup.isdigit():

                student_id = (
                    Validator.student_id(
                        lookup
                    )
                )

                student = (
                    self.repository.get_by_id(
                        student_id
                    )
                )

            else:

                roll = Validator.roll_number(
                    lookup
                )

                student = (
                    self.repository.get_by_roll(
                        roll
                    )
                )

            if student is None:
                raise StudentNotFoundError(
                    "Student record not found."
                )

            TerminalUI.display_student(
                student
            )

            confirmation = TerminalUI.prompt(
                "Type DELETE to permanently "
                "remove this record: "
            ).strip()

            if confirmation != "DELETE":
                TerminalUI.info(
                    "Deletion cancelled."
                )
                return

            self.repository.delete(
                student.id
            )

            TerminalUI.success(
                f"Student '{student.name}' "
                f"was permanently deleted."
            )

        except ValidationError as exc:
            TerminalUI.error(str(exc))

        except StudentNotFoundError as exc:
            TerminalUI.warning(str(exc))

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

        except KeyboardInterrupt:
            TerminalUI.warning(
                "Deletion cancelled."
            )

    # --------------------------------------------------------
    # STATISTICS
    # --------------------------------------------------------

    def show_statistics(self) -> None:

        try:
            stats = (
                self.repository.statistics()
            )

            TerminalUI.section(
                "SUMMARY STATISTICS"
            )

            print(
                f"  Total Students : "
                f"{stats['total']}"
            )

            print(
                f"  Average GPA    : "
                f"{stats['average_gpa']:.2f}"
            )

            top_student = stats[
                "top_student"
            ]

            if top_student:

                print(
                    f"  Top Performer  : "
                    f"{top_student.name} "
                    f"({top_student.gpa:.2f})"
                )

            else:

                print(
                    "  Top Performer  : "
                    "No records"
                )

            print(
                "\n  Grade Distribution:"
            )

            if stats[
                "grade_distribution"
            ]:

                for grade, count in stats[
                    "grade_distribution"
                ].items():

                    print(
                        f"    {grade:<15} "
                        f"{count}"
                    )

            else:

                print("    No data")

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

    # --------------------------------------------------------
    # EXPORT
    # --------------------------------------------------------

    def export_data(self) -> None:

        TerminalUI.section(
            "EXPORT STUDENT DATA"
        )

        choice = TerminalUI.prompt(
            "[1] CSV  [2] JSON: "
        ).strip()

        try:

            students = (
                self.repository.get_all()
            )

            if not students:
                TerminalUI.warning(
                    "There are no records to export."
                )
                return

            if choice == "1":

                path = TerminalUI.prompt(
                    "Output filename [students.csv]: "
                ).strip()

                if not path:
                    path = "students.csv"

                output = (
                    ExportService.to_csv(
                        students,
                        path,
                    )
                )

            elif choice == "2":

                path = TerminalUI.prompt(
                    "Output filename [students.json]: "
                ).strip()

                if not path:
                    path = "students.json"

                output = (
                    ExportService.to_json(
                        students,
                        path,
                    )
                )

            else:

                TerminalUI.error(
                    "Invalid export option."
                )
                return

            TerminalUI.success(
                f"Export completed: "
                f"{output.resolve()}"
            )

        except DatabaseError as exc:
            TerminalUI.error(str(exc))

        except StudentManagerError as exc:
            TerminalUI.error(str(exc))

    # --------------------------------------------------------
    # MENU
    # --------------------------------------------------------

    @staticmethod
    def menu() -> None:

        print(
            """
  +--------------------------------------------------------------+
  |                 STUDENT RECORD MANAGER                       |
  +--------------------------------------------------------------+
  |  1. Add Student                                               |
  |  2. View All Students                                         |
  |  3. View Single Student                                       |
  |  4. Search Students                                           |
  |  5. Update Student                                            |
  |  6. Delete Student                                            |
  |  7. Summary Statistics                                        |
  |  8. Export CSV / JSON                                         |
  |  9. Exit                                                      |
  +--------------------------------------------------------------+
"""
        )

    def run(self) -> None:

        TerminalUI.title(
            "STUDENT RECORD MANAGER"
        )

        TerminalUI.info(
            "SQLite database: "
            f"{self.repository.database.db_path.resolve()}"
        )

        while True:

            try:

                self.menu()

                choice = TerminalUI.prompt(
                    "Select an option [1-9]: "
                ).strip()

                actions = {
                    "1": self.add_student,
                    "2": self.view_all,
                    "3": self.view_single,
                    "4": self.search_students,
                    "5": self.update_student,
                    "6": self.delete_student,
                    "7": self.show_statistics,
                    "8": self.export_data,
                }

                if choice == "9":

                    TerminalUI.success(
                        "Thank you. Goodbye!"
                    )
                    return

                action = actions.get(
                    choice
                )

                if action is None:

                    TerminalUI.error(
                        "Invalid choice. "
                        "Please select 1-9."
                    )
                    continue

                action()
                TerminalUI.pause()

            except KeyboardInterrupt:

                print()

                TerminalUI.warning(
                    "Interrupted. Returning "
                    "to the main menu."
                )

            except EOFError:

                print()

                TerminalUI.info(
                    "Input stream closed. Exiting."
                )
                return

            except Exception as exc:

                TerminalUI.error(
                    f"Unexpected application error: "
                    f"{exc}"
                )


# ============================================================
# COMMAND-LINE ARGUMENTS
# ============================================================

def build_argument_parser() -> argparse.ArgumentParser:

    parser = argparse.ArgumentParser(
        description=(
            "SQLite Student Record Manager"
        )
    )

    parser.add_argument(
        "--db",
        default=DEFAULT_DB,
        help=(
            "SQLite database path. "
            f"Default: {DEFAULT_DB}"
        ),
    )

    parser.add_argument(
        "--export-csv",
        metavar="FILE",
        help=(
            "Export records to CSV and exit."
        ),
    )

    parser.add_argument(
        "--export-json",
        metavar="FILE",
        help=(
            "Export records to JSON and exit."
        ),
    )

    return parser


# ============================================================
# MAIN
# ============================================================

def main() -> int:

    parser = build_argument_parser()
    args = parser.parse_args()

    try:

        database = DatabaseManager(
            args.db
        )

        repository = StudentRepository(
            database
        )

        if args.export_csv:

            students = repository.get_all()

            output = ExportService.to_csv(
                students,
                args.export_csv,
            )

            print(
                f"CSV export completed: "
                f"{output.resolve()}"
            )

            return 0

        if args.export_json:

            students = repository.get_all()

            output = ExportService.to_json(
                students,
                args.export_json,
            )

            print(
                f"JSON export completed: "
                f"{output.resolve()}"
            )

            return 0

        application = StudentApplication(
            repository
        )

        application.run()

        return 0

    except DatabaseError as exc:

        print(
            f"[FATAL DATABASE ERROR] {exc}",
            file=sys.stderr,
        )

        return 1

    except StudentManagerError as exc:

        print(
            f"[ERROR] {exc}",
            file=sys.stderr,
        )

        return 1

    except KeyboardInterrupt:

        print(
            "\nApplication cancelled."
        )

        return 0

    except Exception as exc:

        print(
            f"[FATAL ERROR] Unexpected error: "
            f"{exc}",
            file=sys.stderr,
        )

        return 1


if __name__ == "__main__":
    raise SystemExit(main())